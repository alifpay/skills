---
name: review-sync-transactional
description: Review synchronous HTTP/gRPC APIs that perform transactional operations (payments, transfers, ledger updates, any money-moving action where the client waits for a response). Checks server and context timeout configuration, parameter validation, transaction-date/freshness checks, handling of client disconnect mid-request, context ownership (request context must not be used for DB/broker/provider calls), cascade service state persistence, API versioning discipline (every endpoint is versioned; breaking changes within an existing version are forbidden and must go into a new version; CHANGELOG and deprecation policy required), response code classification, rate limiting per user, commit and rollback endpoints, separation of rollback logic, admin vs public route separation, and background-job offloading of non-transactional work. Trigger when reviewing HTTP/gRPC handlers, controllers, service methods, route definitions, OpenAPI / protobuf / schema files, or any PR that modifies the request/response shape, error codes, status codes, or URL of an existing endpoint.
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
