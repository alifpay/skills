---
name: review-sync-transactional
description: Review synchronous HTTP/gRPC APIs that perform transactional operations (payments, transfers, ledger updates, any money-moving action where the client waits for a response). Checks server and context timeout configuration, parameter validation, transaction-date/freshness checks, handling of client disconnect mid-request, context ownership (request context must not be used for DB/broker/provider calls), cascade service state persistence, API versioning discipline (every endpoint is versioned; breaking changes within an existing version are forbidden and must go into a new version; CHANGELOG and deprecation policy required), response code classification, rate limiting per user, commit and rollback endpoints, separation of rollback logic, admin vs public route separation, background-job offloading of non-transactional work, and server-side idempotency on transactional intake (Idempotency-Key header or client_request_id required, principal-scoped uniqueness, body-hash match for same-key/different-body, cached-response replay on duplicate, concurrent-same-key serialization, documented TTL, record persisted before side effect). Trigger when reviewing HTTP/gRPC handlers, controllers, service methods, route definitions, OpenAPI / protobuf / schema files, idempotency middleware or stores, or any PR that modifies the request/response shape, error codes, status codes, or URL of an existing endpoint.
---

# Synchronous Transactional API Review

## Why this skill exists
Synchronous money-moving endpoints have two hard problems: the client holds the connection open while state mutates, and every disconnect, timeout, or cascade-call failure must leave the system in a consistent state. Junior developers often use the client's request context as the context for DB and downstream calls — when the client disconnects, in-flight commits get cancelled, producing partial state.

## Red flags to catch during review

- [ ] Inbound request `ctx` / `request.Context()` passed directly into DB, broker, or provider calls — **client disconnect aborts the transaction**
- [ ] No server timeout, no per-operation timeout, no context deadline — one slow call can starve the pool
- [ ] Required request parameters not validated before entering business logic
- [ ] No transaction-date / `valid_until` parameter in the request — old requests can replay
- [ ] No check that the transaction date is recent; stale timestamps still processed
- [ ] Response written to the client before the DB commit completes
- [ ] Client disconnect / cancel / timeout not explicitly handled — transaction left in limbo
- [ ] Cascade calls to downstream services with no persisted state — on failure you cannot recover
- [ ] API changes without version bump; error codes, statuses, or shapes changed silently (see **API versioning discipline** below — this is its own required review step)
- [ ] Response codes not classified into Fatal / Pending / Retryable (see `review-error-classification`)
- [ ] Rate limiter absent — double-click / bot replay can create duplicate transactions
- [ ] Mutating transactional POST/PUT does not require a client-supplied idempotency identifier (or accepts one but does not check body-hash on replay, or stores it without principal scope) — see **Server-side idempotency on transactional intake** below
- [ ] Same handler does both commit and rollback logic; no separate cancel endpoint
- [ ] Admin routes (refund, adjust, reverse) exposed on the same public router as user routes
- [ ] Non-transactional side work (emails, push, analytics, audit export) executed inline in the request path instead of enqueued
- [ ] Two users / services can access each other's transactions — authorization check missing

## API versioning discipline

Every public-facing and service-to-service API must be versioned. Breaking changes within an existing version are **not allowed** — any breaking change goes into a new version, and the old version stays alive for a deprecation window. "Just this once, it's a small change" is how client integrations silently break and how mobile apps in the field stop working.

### Rules

- [ ] Every endpoint exposed under an explicit version. Accepted conventions: URL path (`/v1/transactions`), `Accept` header (`Accept: application/vnd.company.v1+json`), or dedicated subdomain (`v1.api.example.com`). Query-parameter versioning (`?version=1`) is discouraged.
- [ ] New services / new endpoints **must** ship under a version from day one. An unversioned endpoint is a block-the-PR finding.
- [ ] **Breaking changes in an existing version are forbidden.** A breaking change goes into a new version (`/v2/...`); the old version stays live for the deprecation window.
- [ ] Each version has a written **CHANGELOG** capturing every change to error codes, statuses, shapes, or semantics, with date + rationale.
- [ ] **Deprecation policy** with a written minimum support window (e.g. 6 months) before an old version is retired. Consumers notified on release of the new version and again before retirement.
- [ ] **CI schema-diff gate** on every PR: compares the new OpenAPI / protobuf / JSON Schema against the deployed version. If the diff contains a breaking change against the same version → build fails.
- [ ] **Version usage telemetry** exists — you must know which versions are still receiving traffic (and from which clients) before retiring anything.

### What counts as breaking (same-version is forbidden — requires a new version)

- Removing a field from the response
- Renaming a field (request or response)
- Changing a field's type (`int → string`, `int32 → int64`, nullable → required, scalar → array)
- Removing an endpoint or changing its URL / method
- Adding a required request parameter without a safe default
- Changing the meaning of an existing enum value
- Tightening validation (previously-accepted input now rejected)
- Changing the authentication scheme
- Changing the HTTP / gRPC status code returned for an existing outcome
- Changing the error code format, numbering, or semantics
- Changing pagination behavior (page-based ↔ cursor-based)
- Changing rate-limit or timeout behavior observable to the client

### Non-breaking (safe in the same version)

- Adding a new optional request parameter **with a safe default** that preserves existing behavior
- Adding a new optional response field (clients must be documented to ignore unknown fields)
- Adding a brand-new endpoint
- Loosening validation (accepting input previously rejected)
- Adding a new enum value **only if** the contract documents that clients must tolerate unknown values — otherwise treat as breaking

### Red flags (versioning-specific)

- [ ] New service or endpoint without a version prefix
- [ ] PR modifies the response shape, adds a required field, renames a field, or changes an enum's semantics **without introducing a new version**
- [ ] PR changes an HTTP / gRPC status code, an error code, or the error body schema for an existing outcome, in the same version
- [ ] PR drops a field from a response in the same version — "nobody uses it" must be verified against version-usage telemetry, not assumed
- [ ] PR changes the request validation to reject input that was previously accepted, in the same version
- [ ] No CHANGELOG entry for a public-API change
- [ ] Deprecation / retirement of an old version without a communicated timeline and without telemetry confirming no remaining consumers
- [ ] No CI schema-diff gate for API contracts
- [ ] Mobile clients in the field that cannot be force-upgraded, combined with a breaking change targeted at the same version they use — this is a guaranteed outage

### PR review steps for versioning

1. Is the modified endpoint under a version prefix / header / subdomain? If no → block.
2. Does the diff modify request shape, response shape, status codes, error codes, enums, or validation of an **existing** endpoint? If yes, run the breaking-change checklist above for each change.
3. If any change is breaking → confirm a new version was introduced and the old version remains unchanged in the diff. If not → block.
4. Confirm the CHANGELOG entry for the version is updated in the same PR.
5. Confirm the CI schema-diff gate ran and passed.
6. For a deprecation / retirement of an old version: confirm version-usage telemetry shows zero (or acceptable) traffic, and that a deprecation notice was sent at least N months prior per policy.

## Server-side idempotency on transactional intake

A money-moving POST / PUT endpoint accepts a client-supplied identifier (`Idempotency-Key` header, `client_request_id` body field, or equivalent). This is the inverse of outbound idempotency (see `review-external-api` → **Idempotency key stability across retries**): instead of being the stable key you *send*, it is the key you *receive* and must honor. Mishandling it produces the same class of incident as outbound mutation — duplicate side effect on retry — but on **your** side, charging your users twice.

### Rules

- [ ] Every mutating transactional endpoint **requires** a client-supplied idempotency identifier on money-moving paths. No identifier → reject the request, do not auto-generate one server-side (a server-generated key cannot match across the client's own retries).
- [ ] The idempotency record stores: **principal** (`client_id` / `api_key_id` / `tenant_id`), **key**, **body hash** over the canonical normalized request body, **status** (`PROCESSING` / `DONE` / `FAILED`), and the **cached response** (status code + body) once terminal.
- [ ] **Body-match check on replay**: same key + different body → reject with a clear error (e.g. HTTP 422). Never accept the new body silently, never re-execute against the new body. *(Money-loss: first call `amount=10` succeeded server-side but timed out on the wire; client retries with `amount=100`. Without body-match, the server either re-executes on the new amount or pretends the old one was the new one — both wrong, one is a charge on the wrong amount.)*
- [ ] **Cached-response replay**: on duplicate key + matching body + record in `DONE`, return the cached response (status code + body) **without re-executing the side effect**. The cached response is part of the idempotency record, not regenerated on each replay.
- [ ] **Principal-scoped uniqueness**: the DB index is `UNIQUE(principal, idempotency_key)`, not `UNIQUE(idempotency_key)`. Two unrelated tenants picking the same key must not collide. (Cross-reference: `review-transaction-model` → unique constraints on business-unique fields.)
- [ ] **Concurrent same-key handling**: when a second request with the same key arrives while the first is still `PROCESSING`, the second request must not start a new side effect. Implement via row-level lock + status flag (the second waits up to a timeout, then returns 409), or return 409 immediately. Either way: **one side effect per (principal, key)**.
- [ ] **TTL on the key**, documented in the API contract (Stripe-style: 24h is a common choice). After expiry the same key represents a new operation. Without a TTL, the table grows forever; without a *documented* TTL, clients cannot reason about long-delayed retries.
- [ ] The idempotency record is **persisted before any side effect** (see `review-transaction-model` → Persist before side effects). On crash between persist and side effect, status is left `PROCESSING`; the next replay or reconciliation resolves through status-check, never by silently re-running.
- [ ] `PROCESSING` records on replay are explicitly handled — the replay waits, returns 409, or routes the client to a status-check endpoint. **Never** treat `PROCESSING` as "not seen yet" and re-execute.

### Red flags

- [ ] Mutating transactional endpoint with **no** idempotency parameter at all (or one that is optional)
- [ ] Idempotency lookup by key alone, no body-hash comparison — client can reuse a key with a different body and get either a duplicate side effect or a silent accept of the wrong body
- [ ] Server auto-generates the idempotency key when the client did not supply one — the key cannot match across the client's own retries, so it provides zero protection
- [ ] On duplicate key with matching body: re-executes the side effect instead of replaying the cached response
- [ ] On duplicate key with matching body: returns a generic "already processed" / 409 without the original response body — client cannot complete its flow without a separate status call
- [ ] Unique index on `(idempotency_key)` only — not scoped per principal. Cross-tenant collision possible
- [ ] Concurrent same-key requests not serialized (no row lock, no status guard) → race produces two side effects (two charges)
- [ ] No TTL on the idempotency table → unbounded growth + ambiguity if a key is reused many months later
- [ ] TTL exists but is undocumented in the API contract → clients cannot reason about safe-retry windows
- [ ] Idempotency record written **after** the side effect → on crash between side effect and record-write, the next replay re-fires the side effect
- [ ] Idempotency record stores only `key + status`, no body hash and no cached response → replay cannot be safely answered
- [ ] `PROCESSING` record on replay treated as "not seen" → second request executes a duplicate side effect while the first is still in flight

### Example — wrong vs right

**Wrong** — key-only lookup, no body match, no cached response, side effect before persistence, concurrent races possible:

```go
func handleCharge(w http.ResponseWriter, r *http.Request) {
    var req ChargeRequest
    json.NewDecoder(r.Body).Decode(&req)

    var existing IdempRow
    if err := db.Get(&existing, "WHERE key = ?", req.IdempotencyKey); err == nil {
        writeJSON(w, 200, map[string]string{"status": "duplicate"}) // ❌ no body match, no real cached response
        return
    }
    resp, _ := provider.Charge(req.Amount, req.CardRef)              // ❌ side effect before persisted record
    db.Save(&IdempRow{Key: req.IdempotencyKey, Status: resp.Status}) // ❌ no principal scope, no body hash, no cached body
    writeJSON(w, 200, resp)
}
```

**Right** — principal-scoped record, body-hash check, cached replay, persisted before side effect, in-flight handled:

```go
func handleCharge(w http.ResponseWriter, r *http.Request) {
    principal := authPrincipal(r)
    req := decodeAndValidate(r)
    bodyHash := canonicalHash(req)

    rec, state, err := idemp.Acquire(srvCtx, principal.ID, req.IdempotencyKey, bodyHash)
    switch state {
    case idemp.BodyMismatch:
        writeJSON(w, 422, errorf("Idempotency-Key reused with a different body"))
        return
    case idemp.AlreadyDone:
        writeJSON(w, rec.CachedStatus, rec.CachedBody) // replay, no side effect
        return
    case idemp.InFlight:
        writeJSON(w, 409, errorf("request in flight, retry with backoff"))
        return
    case idemp.Created:
        // rec.Status = PROCESSING, persisted to DB before any side effect
    }

    resp, err := provider.Charge(srvCtx, req.Amount, req.CardRef, rec.Key)
    final := classifyAndApply(rec, resp, err) // updates rec to DONE/FAILED + caches response
    writeJSON(w, final.StatusCode, final.Body)
}
```

### PR review steps for server-side idempotency

1. Find every mutating transactional POST / PUT in the diff. Confirm each accepts a client-supplied idempotency parameter and rejects requests without one.
2. Find the idempotency lookup. Confirm the index / query is `(principal, key)`, not `(key)`.
3. Confirm a body-hash is stored and checked. Same-key + different-body must reject, not accept.
4. Confirm cached-response replay: matching duplicate returns the original response, does not re-execute the side effect.
5. Confirm concurrent-same-key handling — row lock, status flag, or 409 — anything that prevents two side effects.
6. Confirm the idempotency record is persisted **before** the side effect (cross-reference: `review-transaction-model` → Persist before side effects).
7. Confirm a TTL exists on the idempotency table and is documented in the API contract.
8. Confirm `PROCESSING` records on replay are explicitly handled (wait / 409 / status-check), not treated as unseen.

## What the code SHOULD do

1. Validate every required parameter at the entry point; reject with clear error codes before any mutation.
2. Require a transaction-date / freshness parameter; reject requests older than N seconds/minutes.
3. Use a **server-owned context** (derived from a background context with a per-operation timeout), not the inbound request context, for DB, broker, and provider calls.
4. Configure all three timeouts deliberately: server read/write, handler, downstream-call. Document the values.
5. On client disconnect: detect, stop further downstream calls, and leave the transaction in a recoverable state (PROCESSING with a status-check job enqueued).
6. Persist cascade-call state at each step so the transaction can be resumed or rolled back.
7. Provide explicit **commit and rollback** endpoints; isolate rollback in its own handler.
8. Separate admin routes (internal-only network, stricter auth) from public user routes.
9. Enforce per-user rate limits on money-moving endpoints.
10. Enforce authorization: user A cannot read / modify user B's resources (object-level permission check).
11. Offload non-transactional work (notifications, analytics, audit export) to background jobs via an outbox.
12. Version the API; publish changelog on every change to error codes, statuses, or shapes.
13. On every mutating transactional intake: require a client-supplied idempotency key, persist a record with `(principal, key, body_hash, status, cached_response)` before any side effect, replay the cached response on a matching duplicate, reject mismatched bodies, serialize concurrent same-key requests, and enforce a documented TTL. (See **Server-side idempotency on transactional intake**.)

## PR review checklist

1. Grep every DB / broker / HTTP client call inside the handler. For each: whose context is it using? If it is the request context → flag.
2. Find the response-write line. Confirm the DB commit happened before it, and the response value is read from the committed row.
3. List request parameters; confirm each required one is validated. Confirm a freshness timestamp exists and is checked.
4. Look for separate cancel/rollback endpoint; confirm it exists and is isolated from the main handler.
5. Check the router: are admin routes on a separate mux / network interface?
6. Confirm rate limiter middleware is present on this route.
7. Grep for work that is not transaction-critical (notifications, analytics) inside the handler; flag for background-job extraction.
8. Confirm object-level authorization: the handler checks that the caller owns or has permission for the target resource.
9. Cross-check with `review-error-classification` for the error-branch behavior.
10. Run the **API versioning** sub-checklist (see section above): endpoint versioned? any breaking change? if yes, new version introduced? CHANGELOG updated? schema-diff gate passed?
11. For mutating transactional endpoints, run the **Server-side idempotency on transactional intake** sub-checklist: idempotency key required? `(principal, key)` unique? body-hash stored and checked? cached response replayed on duplicate? concurrent same-key handled? record persisted before side effect? TTL documented? `PROCESSING` replay handled?
