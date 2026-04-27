---
name: review-external-api
description: Review backend code that integrates with external/third-party APIs (payment providers, banks, KYC, card networks, crypto exchanges, SMS/OTP, any outbound HTTP/gRPC to a system you do not own). Checks planning of failure modes, idempotency, timeout and retry strategy, response code classification, duplicate-transaction prevention, two-step auth+confirm flows, amount validation between auth and confirm, transactional-result guarantees, status-check before retry/mark-failed/refund, response schema validation and contract-drift detection (silent type changes, renamed fields, new enum values), refund/reversal guards (never refund on parse error or on self-inferred failure), contract versioning, internal+external ID uniqueness, reconciliation, and deployment of version bumps to a live integration (blue-green, staged rollout, shadow/parallel-run with no side effects, one-toggle rollback rehearsed in staging, stable idempotency-key format across the cutover, drain wait by oldest in-flight transaction, tightened reconciliation cadence). Trigger when reviewing clients, adapters, SDKs, webhooks, response parsers, refund/reversal handlers, release plans for integration bumps, or any code that calls an external provider.
---

# External API Integration Review

## Why this skill exists
Integrations with external systems are where money leaks. The external API owner's failure modes become your failure modes, but the responsibility for consistency stays with you. Most production incidents come from unplanned error branches, missing idempotency, or trusting the external API's response shape without validation.

### Real incidents this skill is calibrated against

1. **Redis-error-as-FAILED.** Infra call failed, code set the transaction to `FAILED` locally; when infra recovered, the flow replied `SUCCESS` upstream. Local ledger and upstream state diverged. (See `review-error-classification`.)
2. **Silent contract drift + refund.** The provider changed a response field type (e.g. `amount` from integer to string, or an ID field from `int` to `long`) without notifying us. Our parser threw. The code treated parse failure as a **final failure** and triggered the **refund/reversal** path — so we returned money to the user. But on the provider side, the transaction had completed successfully. Net result: the customer kept the goods/service, the provider kept our money, we paid the refund out of our own pocket. **Parse failure is never proof of a failed transaction. Never refund without a confirmed failure from the authoritative source.**
3. **Retry mutated the transaction ID → double charge.** On a retry path, the service appended a random number to the original transaction ID before sending to the provider:
   ```go
   if dbStatus == "retry" {
       transferID += "_" + utils.GenRandNum()  // ❌
   }
   ```
   Every retry looked like a brand-new transaction to the provider. The provider's own idempotency was bypassed (different ID = different transaction), and if the first attempt had already succeeded at the provider but we failed to read the response, the retry created a **second real charge** on the same user for the same logical operation. Reconciliation was also broken — the provider's records used the original ID, our DB and later retries used mutated IDs, and they could not be matched.
   **The idempotency key must be stable across every retry of the same logical operation. A retry is the same request, not a new one.**

## Before integration starts
- Is the provider's documentation read and summarized in the PR / design doc? Are all error codes, timeouts, and edge cases enumerated?
- Is there a written error-handling and retry strategy? Vague "we'll retry on errors" is insufficient.
- Does the provider have known duplicate-transaction issues or unknown-response issues? If yes → **do not integrate until resolved with the provider.** Block the PR.
- Is there a contractual agreement covering **versioning, deprecation notice, and backward compatibility**? Without a written commitment to notify before breaking changes, assume the provider will change the response shape silently and design for it.
- Is there a plan to detect contract drift (schema validation + periodic smoke test) independent of the provider's announcements?

## Idempotency key stability across retries

The idempotency key (transaction ID, external reference, `Idempotency-Key` header — whatever the provider uses to deduplicate) **must be stable across every retry of the same logical operation**. A retry is the same request re-sent; it is not a new request.

Mutating the key on retry defeats the entire purpose of idempotency and is one of the most direct ways to cause a double-charge: if the first attempt succeeded on the provider side but we couldn't read the response, a mutated-key retry is a brand-new transaction to the provider and charges the user a second time.

### Rules

- [ ] The idempotency key is generated **once**, when the transaction row is first created, and **persisted on the row**.
- [ ] Every subsequent outbound attempt (original + all retries) reads the key back from the row and sends **byte-identical** to the original.
- [ ] The payload sent on retry is byte-identical to the original (same amount, same currency, same references) unless the provider's retry protocol explicitly requires a different value — document any such exception.
- [ ] When the provider rejects with "duplicate transaction ID" (or equivalent): **do not mutate the ID**. Call the provider's **status-check** endpoint to learn whether the original attempt succeeded, then act on that result. See `review-error-classification` and refund guards above.
- [ ] Retry logic lives in one place (a retry policy / executor), and that place does not touch the identifier or the payload.
- [ ] The idempotency key is long enough + random enough to be globally unique without needing any mutation (UUID v4 or equivalent). If the key is short / predictable, fix the generation, not the retry.

### Anti-patterns to catch

- [ ] Appending a random number, timestamp, counter, or nonce to the ID on retry:
  ```go
  if dbStatus == "retry" { transferID += "_" + utils.GenRandNum() }       // ❌
  if attempt > 0         { transferID = fmt.Sprintf("%s-%d", id, attempt) } // ❌
  if retry               { transferID = uuid.NewString() }                 // ❌
  ```
- [ ] Regenerating a UUID / new ID inside the retry loop instead of reading the persisted one
- [ ] Using a timestamp, request timestamp, or `now()` as part of the idempotency key (non-deterministic across retries)
- [ ] Hashing the payload with a time-dependent component into the key
- [ ] "Bypassing" a provider's duplicate-ID rejection by changing the ID — this is **always** wrong; the correct response is a status-check
- [ ] Different parts of the codebase deriving the idempotency key differently (some from the DB row, some from the request, some from an in-memory computation)
- [ ] Idempotency key not persisted on the transaction row → no way to guarantee reuse across retries / process restarts
- [ ] Retry executor that modifies the payload (rounding, reformatting, re-signing with a new nonce) between attempts

### Example — wrong vs right

**Wrong** — retry mutates the ID; a prior-succeeded attempt becomes a second charge:

```go
func chargeWithRetry(tx Transaction) error {
    transferID := tx.ID
    for attempt := 0; attempt < 3; attempt++ {
        if attempt > 0 {
            transferID = tx.ID + "_" + utils.GenRandNum() // ❌ new ID each try
        }
        resp, err := provider.Charge(transferID, tx.Amount)
        if err == nil { return apply(resp) }
    }
    return errors.New("exhausted retries")
}
```

**Right** — stable key from the persisted row; duplicate-ID response triggers status-check, not mutation:

```go
func chargeWithRetry(tx Transaction) error {
    // tx.IdempotencyKey was generated and persisted when the row was created.
    for attempt := 0; attempt < 3; attempt++ {
        resp, err := provider.Charge(tx.IdempotencyKey, tx.Amount)
        switch classify(err, resp) {
        case Fatal:     return apply(resp)                   // authoritative failure
        case Retryable: continue                              // connection refused, 5xx no-effect
        case Duplicate: return reconcile.FromStatusCheck(tx)  // provider says "already seen" → ask for status, never mutate ID
        case Pending, Unknown:
            queue.EnqueueStatusCheck(tx.ID)                  // leave PROCESSING, do not retry immediately
            return nil
        }
    }
    queue.EnqueueStatusCheck(tx.ID)
    return nil
}
```

## Response contract validation and drift detection

Treat every response from an external provider as potentially-changed since the last deploy. The provider may rename fields, change types (`int → string`, `int32 → int64`, nullable → missing, object → array), add new enum values, or reorder elements — without telling you.

### Rules
- [ ] Every field read from the response is validated: **presence**, **type**, and **value range** checked before use. Do not pass the raw parsed object into business logic.
- [ ] Validation failure produces an **Unknown** classification, never Fatal. (Classification rules: see `review-error-classification`.)
- [ ] On validation failure: persist the **raw response body** for forensics, alert on-call, leave the transaction in `PROCESSING`, enqueue a status-check. Do not mark FAILED. Do not refund.
- [ ] New / unknown enum values in a known field → Unknown, not Fatal. The provider adding `PENDING_REVIEW` to a status enum you only know `APPROVED`/`REJECTED` for must never be silently interpreted as one of the known values.
- [ ] Schema defined centrally (JSON Schema, protobuf, typed struct with explicit decoder). Ad-hoc `map[string]any` / `dict` extraction is a red flag.
- [ ] Periodic **smoke / contract test** against the provider's sandbox in CI — parses a known-good response and fails the build when the shape drifts, so you see the change before production traffic does.
- [ ] Raw response logged (with sensitive data redacted) for every transaction, so post-incident reconstruction is possible.

## Refund and reversal guards

A refund or reversal on the wrong trigger loses real money. These paths need stricter controls than the forward path.

### Rules
- [ ] **Refund / reversal is never triggered by a parse error, timeout, connection error, or any self-inferred failure.** Only by (a) an authoritative failure response from the provider, (b) an explicit result from a status-check call, or (c) a manual back-office action with audit trail.
- [ ] Every refund/reversal call sites **calls the provider's status-check endpoint first** and confirms the transaction actually failed or does not exist on the provider side.
- [ ] Refund/reversal is idempotent and carries its own idempotency key distinct from the original transaction key.
- [ ] Refund/reversal logic lives in a **separate handler** from the forward path (see `review-sync-transactional`).
- [ ] Automated refund paths (e.g. "on failure, auto-refund") are disabled by default; enabling requires an explicit design review.
- [ ] Every refund/reversal produces an alert + audit record with: trigger source, provider status at time of decision, and operator (or "system").

## Deploying a version bump to a live integration

Why: A live integration is part of your runtime contract with the provider. A version bump — yours, theirs, or both — can change response shape, error semantics, signing, ID format, or timing. A "deploy and watch" rollout converts every incompatibility into a user-impacting failure that costs real money to unwind. Money-moving integrations ship version bumps with a written release plan, a staged rollout, and a one-toggle rollback, every time.

This applies whether the change is **on your side** (new client code, new parser, new request shape) or **on the provider's side** (the provider has announced a contract bump and you must integrate against it).

### Rules

- [ ] **Written release plan** in the PR or runbook: what changed (your side and provider's side), what can break, what dashboards to watch, what triggers rollback, who is on-call during the window. "Deploy and watch logs" is not a plan.
- [ ] **Staged rollout** behind a feature flag or routing toggle: e.g. 1% → 10% → 50% → 100%, with a hold at each step long enough for the slowest provider response to come back at least 2× over (so a slow tail does not look like silence).
- [ ] **Blue-green at the integration boundary**: old code path and new code path coexist and are switched by a config flip, not a redeploy. Both paths read/write the same DB and share the same idempotency-key space — see "External-ref / idempotency-key stability" below.
- [ ] **Parallel-run / shadow comparison** for shape-changing bumps: route the same logical request through the new code path in shadow mode, compare its parsed result against the old path's result, fail rollout if responses diverge beyond an agreed threshold. **The shadow path produces zero side effects** — does not call the provider a second time, does not write a second outbox row, does not bill twice.
- [ ] **Rollback is one toggle**, executable by on-call without a release engineer. Not "git revert and redeploy". Decision-to-rollback latency must be measured in seconds, not minutes.
- [ ] **Rollback rehearsed in staging** within the last release cycle. An untested rollback fails at the moment it is needed.
- [ ] **Pre-deploy gate**: contract / smoke test against the provider's sandbox passes on the new code against the new response shape (cross-reference: **Response contract validation and drift detection**).
- [ ] **Post-deploy watch list** is identified before rollout: error rate, Unknown-sub-kind rates (`null_status`, `unparseable`, `unknown_enum` — see `review-error-classification`), p99 latency, reconciliation divergence count. Spike on any of these = stop rollout.
- [ ] **Reconciliation cadence tightened** during the rollout window — divergence is expected to spike, so the reconciliation job runs more frequently and pages on threshold breach instead of waiting for the regular window.
- [ ] **External-ref / idempotency-key format stable across the version boundary** unless renegotiated with the provider in writing. Changing the key format breaks matching for in-flight transactions and for historical reconciliation across the cutover.
- [ ] **Drain wait before retiring the old path**: measured against the actual oldest `PROCESSING` row that started on the old code, not a wall-clock guess. Retries and status-checks for old-path transactions must keep working until that row resolves.
- [ ] **Provider-side bumps**: confirm the switchover window with the provider in writing, schedule against it, and define handling for transactions that span the boundary (started on the old contract, expected to confirm on the new). Status-check responses across the cutover get explicit test coverage.
- [ ] **Rollout scheduled inside the on-call coverage window**, not over a weekend or off-hours unless a dedicated on-call is assigned for the duration.

### Red flags

- [ ] Version bump shipped 100% of traffic in one deploy — no flag, no toggle, no staging fraction
- [ ] Rollback plan is "git revert and redeploy" — latency too high for a money-moving path
- [ ] No shadow / parallel-run for a response-shape change — first signal of incompatibility is a user-impacting failure
- [ ] Shadow path that **calls the provider** (creates a second real charge per request) — turns the rollout itself into a double-charge incident
- [ ] Watch dashboards not chosen pre-deploy — on-call has nothing concrete to read during the window
- [ ] Old code path removed in the same deploy that adds the new one — no drain
- [ ] Old code path retired by wall-clock timer instead of by oldest-`PROCESSING` age — in-flight transactions get stranded
- [ ] Provider-side bump deployed without a written switchover window from the provider
- [ ] External-ref / idempotency-key format changed silently at the version boundary, breaking match for in-flight transactions and for reconciliation across the cutover
- [ ] Reconciliation cadence unchanged during rollout — divergence can build for hours before it is noticed
- [ ] Rollback toggle exists but has not been exercised in staging in the current release cycle
- [ ] Rollout scheduled outside the on-call coverage window, or with no named on-call for the duration
- [ ] Pre-deploy contract test not run against the provider's sandbox on the new code

## Red flags to catch during review

- [ ] Outbound call sent **before** persisting the transaction row — on failure you have no record to reconcile
- [ ] Missing idempotency key on the outbound call, OR key generated after the call, OR key not reused on retry
- [ ] Idempotency key / transaction ID **mutated between retries** — random suffix, timestamp, counter, UUID regenerated per attempt. **Classic double-charge bug** (see **Idempotency key stability across retries** below).
- [ ] Response "duplicate transaction ID" from the provider handled by mutating the ID and retrying, instead of calling status-check
- [ ] No uniqueness constraint binding your internal ID to the provider's external ID (invites duplicates)
- [ ] Response parsing with no validation — assumes JSON/XML structure is always as documented
- [ ] No branch for empty response body, missing fields, unknown status codes, type changes (`"123"` vs `123`, `int` vs `long`, nullable → missing, scalar → array)
- [ ] Unknown enum value in a known field silently coerced to a known one (e.g. unrecognized status → treated as `APPROVED` or `FAILED` by default branch)
- [ ] Parse failure / schema-validation failure routed to a final state (`FAILED`, `ERROR`) instead of `PROCESSING` + status-check — **this is the money-refund-on-parse-error trap**
- [ ] **Refund / reversal triggered by any self-inferred failure** (parse error, timeout, infra error). Must only be triggered by an authoritative provider response or a status-check result.
- [ ] Raw response body not logged — no way to reconstruct what the provider actually returned after the fact
- [ ] Schema defined inline / ad-hoc (`map[string]any`, `dict`, untyped JSON) instead of a centralized typed decoder
- [ ] No contract / smoke test running against the provider's sandbox on a schedule or in CI
- [ ] Retry loop on Unknown/unparseable responses instead of Circuit Breaker (retries create duplicates)
- [ ] Retry, mark-failed, **or refund** without first calling the provider's status-check endpoint
- [ ] Webhook / callback handler writes the transaction status with a blind `UPDATE ... WHERE id = ?` — no current-state guard. **A late callback can overwrite a final status; if refund is wired to status change, money is returned on a successful transaction.** Status updates from webhooks must be guarded by current state + stale-timestamp check (see `review-transaction-model` → Final-state immutability).
- [ ] Webhook handler not idempotent — replays change state or re-trigger side effects
- [ ] Two-step auth + confirm flow where confirm does not re-validate that the amount equals the authorized amount (**attacker can tamper with confirm amount**)
- [ ] Two-step flow where authorization and confirmation can race or be re-ordered
- [ ] Integration result not wrapped in a DB transaction — partial state possible on failure
- [ ] No reconciliation job comparing local records with provider records
- [ ] No alerting + manual-review record on Unknown responses
- [ ] Version bump to an existing integration deployed without blue-green / staged rollout, without a one-toggle rollback, without a shadow / parallel-run for shape changes, or without a tightened reconciliation cadence during the window (see **Deploying a version bump to a live integration**)
- [ ] Cross-reference with `review-error-classification` — infra errors on the outbound call must not become final FAILED status

## What the code SHOULD do

1. Persist the transaction locally with a **unique internal ID** and the provider's **external ID** (or placeholder) before calling out.
2. Send a stable idempotency key; reuse it on every retry attempt.
3. Classify every response into Fatal / Retryable / Pending / Unknown (see `review-error-classification`).
4. Validate response shape with a **centralized, typed schema** before use: required fields present, types as expected, status code / enum in a known set. Unknown values → Unknown classification, not silent coercion.
5. On parse / schema-validation failure: keep transaction `PROCESSING`, log the raw response body (redacted), alert, enqueue status check. Never transition to FAILED. Never refund.
6. Call the provider's status-check endpoint before retrying, marking a transaction FAILED, **or issuing a refund / reversal**.
7. Run a periodic contract / smoke test against the provider's sandbox so drift is caught in CI before it reaches production traffic.
8. For two-step flows: persist the authorized amount and re-compare it on confirm; reject or alert on mismatch.
9. Wrap the full integration outcome in a DB transaction so either all local effects commit or none do.
10. Run a periodic reconciliation job comparing local state vs. provider state; alert on divergence.
11. Apply Circuit Breaker to Unknown responses; do not retry.
12. Keep refund / reversal in a separate handler with its own idempotency key; refund is triggered only by authoritative failure or status-check result, never by a self-inferred failure.
13. Deploy version bumps via blue-green / staged rollout with a written release plan, a one-toggle rollback, a shadow / parallel-run for shape changes (no side effects on the shadow path), a stable external-ref / idempotency-key format across the cutover, a tightened reconciliation cadence during the window, and a drain wait gated on the oldest in-flight `PROCESSING` row before retiring the old path.

## PR review checklist

1. Find the outbound call in the diff. Confirm the local row was persisted before it.
2. Grep for the idempotency key generation. Confirm it is persisted with the row and reused on retry.
   - Inspect every call site of the outbound call: is the key read from the persisted row, or recomputed? Recomputed → flag.
   - Grep the diff for string concatenation against the ID / key on retry paths (`+ "_"`, `+ strconv.`, `fmt.Sprintf("%s-%d"`, `uuid.NewString()` inside a retry loop, `Rand`, `GenRandNum`, `Nonce`, `time.Now().Unix()` used in the key). Any of these → block.
   - Inspect the "duplicate transaction ID" branch from the provider: correct behavior is status-check, not ID mutation.
3. Find the response decoder. Is it a centralized typed schema, or ad-hoc map/dict extraction? If ad-hoc → flag.
4. For every field extracted from the response: is presence + type + value range checked? Unknown enum values handled as Unknown (not silently coerced)?
5. Find the error branch for parse / schema-validation failure. Does it mark the transaction FAILED? If yes → block. Correct behavior: keep PROCESSING, log raw body, alert, enqueue status check.
6. Grep the diff for `refund`, `reverse`, `rollback`, `chargeback`, `cancel`. For every call site: is the trigger an authoritative provider response or status-check result? Or is it a parse error / timeout / infra error? If the latter → block.
7. List every status code / error code handled. For each: is the classification correct? Are Unknown codes routed to alert + circuit breaker, not retry?
8. For auth+confirm integrations: grep for the amount comparison on confirm. If absent → block.
9. Confirm the local DB commit and the outbound-call result are in a single transactional boundary.
10. Is there a reconciliation job? Check its schedule and what it compares.
11. Is there a contract / smoke test against the provider's sandbox? When was it last updated?
12. For a version change to an existing integration (yours or the provider's): walk **Deploying a version bump to a live integration** top-to-bottom — written release plan, staged rollout, shadow / parallel-run with no side effects, one-toggle rollback rehearsed in staging, stable external-ref / idempotency-key format, tightened reconciliation cadence, drain wait by oldest `PROCESSING` age, on-call coverage. Any missing item → block.
