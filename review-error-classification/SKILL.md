---
name: review-error-classification
description: Review backend error handling in transactional/financial flows to prevent money loss from misclassified failures. Checks that infrastructure errors (Redis, RabbitMQ, Kafka, database, HTTP timeout, upstream service failure, context cancellation) are NOT converted into final FAILED status, that unknown/transient errors leave the transaction in PENDING/PROCESSING with alerting, that the response returned to the caller matches the locally-committed state (never reply SUCCESS when the local commit or downstream confirmation is missing), that a status-check precedes any retry or mark-failed, and that a failure to write to the local database trips an inbound circuit breaker (halts new transactional traffic) and pages on-call rather than marking the row FAILED. Trigger when reviewing catch blocks, error branches, status transitions, response writers, DB-write error handling, or any code path that updates transaction state in payment, wallet, ledger, or transactional services.
---

# Error Classification Review

## Why this skill exists — real incident

A developer deployed a service without proper review. Redis was temporarily unavailable, so the code entered its error branch and marked the transaction as **FAILED** locally. When Redis recovered, the same flow continued and returned **SUCCESS** to the upstream caller. Internal ledger said FAILED. Upstream service recorded SUCCESS. Money was lost reconciling the gap.

**Core rule for this codebase: infrastructure failures are not business failures. Unknown outcomes are not final failures. Only the authoritative source (provider, bank, host, or verified status check) may move a transaction to a final state.**

## Classify every error into exactly one of four buckets

| Category | Meaning | Allowed action |
|---|---|---|
| **Fatal** | Confirmed business/permanent reject from the authoritative source (bad PAN, insufficient funds, hard decline) | May mark `FAILED` |
| **Retryable** | Outcome known NOT to have happened (connection refused before request was sent, DNS failure, 5xx with no side effect) | Retry with backoff |
| **Pending** | Outcome unknown — request was sent but the response was not received intact (timeout after send, socket reset mid-response, broker ack missing) | Keep `PROCESSING`, schedule status check |
| **Unknown** | A response came back but cannot be trusted (see sub-kinds below) | Keep `PROCESSING`, alert **with sub-kind**, open circuit breaker |

A Redis/RabbitMQ/Kafka/DB/HTTP error is **never** Fatal on its own. It is Retryable, Pending, or Unknown — based on whether the downstream request was sent and whether the outcome can be confirmed.

### Unknown has named sub-kinds — do not lump them together

"Unknown" is not a single bucket for logs / alerts / metrics. Each of the following is a distinct root cause and must have its own alert name, metric label, and reconciliation ticket so that investigation can separate them from each other and from Pending (timeouts):

| Sub-kind | Symptom | Why it is not the same as timeout |
|---|---|---|
| `null_status` | Response arrived, but the status / result-code field is `null` / empty string / missing | The provider accepted the request and replied — the outcome exists on their side but was not reported to us. Timeout did not happen. |
| `empty_body` | HTTP 200 (or similar) with zero-length body | The transport succeeded; the payload is wrong. Distinct from both timeout and null-status. |
| `unparseable` | Body present but cannot be decoded (invalid JSON/XML, schema mismatch) | Response was received; parser rejected it. Often a sign of silent contract drift (see `review-external-api`). |
| `unknown_enum` | A known field carries a value we do not recognize (new status the provider added without notice) | Response parsed fine; semantics are new. |
| `unknown_id` | The `id` / external-ref field is missing, malformed, or the wrong type | We may have succeeded at the provider but cannot correlate back to our row. |
| `unexpected_code` | HTTP / gRPC / business status code outside every documented value | Response classification cannot be decided. |
| `signature_fail` | Response signature / HMAC / JWT could not be verified | Body may be authentic and we just cannot verify, or may be tampered. Treat as Unknown, alert, never Fatal. |

Rules:
- [ ] **`null_status` (and friends) are separated from `timeout` in logs, metrics, and alerts.** Timeout is Pending; null-status is Unknown. Do not route them both into one generic `error` / `unknown_error` label — you will lose the signal needed to debug and to decide retry/circuit-breaker policy.
- [ ] Each sub-kind gets its **own alert name** (`provider_null_status`, `provider_empty_body`, `provider_unparseable`, etc.) and its own metric label.
- [ ] Each sub-kind leaves the transaction in `PROCESSING` and enqueues a **status-check**, but the investigation ticket it produces is labelled so the on-call engineer can tell *why* it went to Unknown without reading raw logs.
- [ ] A default / catch-all Unknown handler that emits a single generic alert is a red flag — it means sub-kinds are being lost at the point of classification.
- [ ] Circuit breaker policy may differ per sub-kind: a sustained rate of `null_status` / `unparseable` often indicates a provider deploy or contract drift and should trip the breaker faster than raw network timeouts.

## Local DB failure — halt new traffic, not just alert

A failure to commit to **your own** database is structurally different from a failure to reach a provider. The provider path has a defined recovery (PROCESSING + status-check + reconcile). A failed local commit means the source of truth is unreachable: you cannot honor *persist-before-side-effects* (see `review-transaction-model`), you cannot record what you just attempted, and every next inbound transaction will hit the same wall. The correct response is **fail-stop on the inbound side**.

### Rules

- [ ] On a DB-write failure on a transactional path, the row is **not** transitioned to `FAILED` — same rule as for any other infra error. It stays `PROCESSING`, or was never persisted in the first place (which the alert + post-incident reconciliation must catch).
- [ ] DB-write failure trips an **inbound circuit breaker** on transactional endpoints: subsequent transactional requests are rejected with a clear "service degraded" status (e.g. HTTP 503 with `Retry-After`) until the breaker closes. Read-only / status-check / reconciliation / back-office paths stay open so on-call can see in and drain backlog from the outside.
- [ ] On-call **pages immediately** on the first DB-write failure on a transactional path — not on a per-minute aggregate, not buffered. Money paths cannot wait for an aggregation window.
- [ ] The breaker has both an **automatic close criterion** (e.g. N consecutive successful writes on a probe path within a window) and a **manual override**, so operators can force open or force closed.
- [ ] After the breaker closes, a **runbook exists** for finding rows left mid-flight, comparing each against any external side effect that may have started, and resolving via reconciliation.
- [ ] DB failures are labelled distinctly in metrics + alerts: `db_connection_refused`, `db_timeout`, `db_deadlock`, `db_constraint_violation`. A unique-constraint violation is *idempotency working as designed* and must not trip the breaker; a connection failure must.

### Red flags

- [ ] DB error in a transactional handler caught and returned as a generic 500 — no page, no breaker, no row-state guarantee; the next request hits the same wall
- [ ] DB-write failure transitions the row to `FAILED` (same class as Redis-error-as-FAILED — see incident at top of skill)
- [ ] Handler proceeds to call the provider after the local persist failed — violates persist-before-side-effects (see `review-transaction-model`)
- [ ] Single generic "DB error" alert with no separation between connection failure / timeout / deadlock / constraint violation — operator cannot decide what to do
- [ ] Inbound breaker that closes purely on a wall-clock timer with no success-probe — re-opens the gate while the DB is still degraded
- [ ] Inbound breaker that also blocks the status-check / reconciliation / back-office paths — operators lose the ability to drain backlog
- [ ] Constraint-violation errors trip the breaker — every legitimate idempotent retry shuts down inbound traffic

### Example

**Wrong** — DB error swallowed, request continues to the provider; inbound stays open and the next request hits the same failure:

```go
if err := db.Save(&tx); err != nil {
    log.Error("save failed", err)            // ❌ no page, no breaker
}
resp, _ := provider.Charge(...)              // ❌ side effect with no persisted row
return resp
```

**Right** — DB-write failure pages, trips inbound breaker, refuses to call the provider:

```go
if err := db.Save(srvCtx, &tx); err != nil {
    alerts.Page("db_write_failed", classify(err), tx.ID, err)
    inboundBreaker.RecordFailure()
    return ErrServiceDegraded                // 503 + Retry-After, no provider call
}
```

## Red flags to catch during review

- [ ] `catch` / `if err != nil` / `except` branch on **Redis, RabbitMQ, Kafka, database, HTTP client, gRPC** that sets status to `FAILED` / `CANCELED` / any final state
- [ ] `context.Canceled` / `context.DeadlineExceeded` / `ctx.Err()` → `FAILED` (the context belonged to the client request, not to the transaction)
- [ ] Any `PROCESSING → FAILED` transition NOT driven by an authoritative response from the provider
- [ ] Response written to the caller **before** the local DB commit succeeds
- [ ] Retry loop on Unknown / unparseable responses (should be circuit breaker, not retry — retries can create duplicates)
- [ ] No status-check call before retry or mark-failed
- [ ] Infra error path only logs — no alert, no row left for manual reconciliation
- [ ] DB-write failure on a transactional path that does NOT page on-call AND does NOT trip an inbound circuit breaker — the next request hits the same wall and unresolved `PROCESSING` rows pile up (see **Local DB failure — halt new traffic, not just alert**)
- [ ] `null` / empty / missing response status lumped with `timeout` into one generic "error" / "unknown_error" label in logs, metrics, or alerts — each Unknown sub-kind must have its own label (see **Unknown has named sub-kinds**)
- [ ] Generic `default:` / `else:` branch catches every non-success case and produces one alert — sub-kinds are lost at classification time
- [ ] `null` status silently coerced to `FAILED` or `APPROVED` — must be `Unknown`, status-check, alert
- [ ] State machine silently allows `PROCESSING → FAILED` without guarding on the source of the transition
- [ ] Missing idempotency key on outbound call OR idempotency key generated AFTER the outbound call
- [ ] Same handler does both the request AND the cancel/rollback path (rollback must be isolated)
- [ ] Using the inbound HTTP request's `ctx` (or equivalent) as the context for DB / broker / provider calls — client disconnect will abort in-flight work
- [ ] Partial success treated as full success (e.g. DB commit ok but broker publish failed, and caller still gets `SUCCESS`)
- [ ] Two sources of truth for the same transaction (e.g. Redis cache marked done but DB not committed, and the response uses the cache)

## What the code SHOULD do

1. Persist the transaction row with status `NEW`/`PROCESSING` **before** any outbound call.
2. Generate and persist an idempotency key before the outbound call; send it **byte-identical** on every attempt. Never append a random suffix / timestamp / counter to the key on retry — that makes each retry a new transaction to the provider and causes double-charge (see `review-external-api` → **Idempotency key stability across retries**).
3. On infra error, timeout, or unknown response: **leave status as `PROCESSING`**, enqueue a status-check job, optionally open a circuit breaker. Do not mark `FAILED`.
4. On a failure to write to your own DB on a transactional path: keep the row out of any final state, do **not** call the provider, page on-call immediately, and trip an **inbound circuit breaker** that rejects new transactional requests until DB writes are healthy. Status-check / reconciliation / back-office paths stay open. (See **Local DB failure — halt new traffic, not just alert**.)
5. Only move `PROCESSING → APPROVED / FAILED` on:
   - an authoritative response from the provider, OR
   - the result of an explicit status-check call, OR
   - an explicit back-office/manual reconciliation action.
6. Response to the caller must reflect the locally-committed state. If the commit did not succeed, the response cannot be `SUCCESS`.
7. Isolate rollback/cancel logic in a separate handler.
8. Use a server-owned context (not the inbound request context) for DB and downstream calls.
9. Alert on every Unknown / Pending branch and leave a reconciliation record.

## PR review checklist

1. Grep the diff for `redis`, `rabbit`, `kafka`, `http.Client`, DB driver calls, `ctx.Err`, `DeadlineExceeded` inside any transactional path.
2. For every error branch that touches transaction status, ask: *"is the outcome confirmed by the authoritative source?"* If not → the branch must not write a final status.
3. Find the DB-write error branch on every transactional path. Confirm: (a) the row is not moved to `FAILED`, (b) the handler does not proceed to a provider call after the local persist failed, (c) an immediate page is raised, (d) an inbound circuit breaker is tripped (or there is a documented reason it is not). If any of those is missing → block.
4. Locate every response writer (JSON reply, gRPC return, event publish). Confirm the local commit happened first and that the response value is derived from the committed row, not from an in-memory flag.
5. Inspect the state machine. List every transition ending in a final state and confirm each has an authoritative-source guard.
6. Check alerting: every Pending / Unknown branch must produce an alert + durable record. Every **Unknown sub-kind** (`null_status`, `empty_body`, `unparseable`, `unknown_enum`, `unknown_id`, `unexpected_code`, `signature_fail`) must have its own distinct alert name and metric label — a single generic "unknown_error" alert is a finding.
7. Grep the response-classification code for conditions like `resp.Status == nil`, `resp.Code == ""`, `len(body) == 0`, parser `!ok` / error-decode branches. Confirm each routes to its **named** sub-kind, and not into the same branch as `timeout` / `DeadlineExceeded`.
8. Check idempotency: the outbound call must carry a key that is persisted before the call and reused on retry.
9. Check context ownership: DB / broker / provider calls must not use the inbound-request context.
10. Check the rollback handler: is it separate from the main request handler? Does it leave the transaction in a consistent state on partial failure?

## Example — wrong vs right (Go)

**Wrong** — Redis error becomes a final failure; caller may still get SUCCESS later:

```go
if err := redis.Set(ctx, key, val, ttl).Err(); err != nil {
    tx.Status = "FAILED"            // ❌ infra error → final state
    db.Save(&tx)
    return resp{Status: "FAILED"}, nil
}
// ... continues and later writes SUCCESS to caller
```

**Right** — infra error is Pending, not final; caller sees the truth:

```go
if err := redis.Set(srvCtx, key, val, ttl).Err(); err != nil {
    // Unknown whether downstream effects happened — keep PROCESSING
    alerts.Raise("redis_set_failed", tx.ID, err)
    queue.EnqueueStatusCheck(tx.ID)
    return resp{Status: "PENDING"}, nil   // caller told the truth
}
```
