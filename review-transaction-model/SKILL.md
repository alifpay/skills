---
name: review-transaction-model
description: Review the data model, state machine, timestamps, and integrity rules for transaction/ledger/payment entities. Checks allowed state transitions (NEW → PROCESSING → APPROVED/FAILED/CANCELED/REVERSED/ERROR → FINAL), final-state immutability enforced at the DB level (late or stale callbacks/webhooks must never overwrite a final status — every UPDATE must be guarded by the current state), refund/reversal never triggered by a status-overwrite attempt, timestamp semantics (CREATED_AT, COMPLETED_AT, LAST_UPDATED_AT, UTC, precision, retention), monetary storage in minor units, immutability of core transaction fields, exchange-rate and fee handling (volatility, fallback source, expiration), primary/foreign/unique key design, single-table storage, ACID compliance, master-connection-only for writes, persist-before-outbound-call rule, isolated rollback handlers, object-level authorization, timeout configuration, commit/rollback coverage, and connection closing. Trigger when reviewing transaction/order/ledger schemas, migrations, ORM models, state-transition code, status-update handlers, webhook/callback handlers, or any code touching financial entity fields.
---

# Transaction Data Model Review

## Why this skill exists
The transaction model is the source of truth for money. Wrong state transitions, mutable amounts, floating-point math, or missing uniqueness constraints directly cause double-spends, lost funds, and unreconcilable ledgers. These bugs survive code review because the individual diffs look innocent — only the model-wide view catches them.

### Real incident this skill is calibrated against

**Late callback overwrote a final status, refund fired.** A transaction reached `APPROVED` (final). Some time later, another service / external callback arrived with a `FAILED` status for the same transaction ID. The status-update handler did a plain `UPDATE status=...` with no current-state guard, so `APPROVED` was overwritten to `FAILED`. The overwrite triggered the refund/reversal path, and money was returned to the user — even though the transaction had actually completed successfully.

**Rules this forces:**
- Final statuses are immutable. Not "usually". Not "unless the callback is authoritative". **Immutable, enforced at the database level.**
- Every status UPDATE must be guarded by the current state in the WHERE clause — never a blind `UPDATE ... WHERE id = ?`.
- Late / stale / conflicting callbacks are **reconciliation events**, not direct updates. They are logged, alerted, and routed to manual review — never silently applied.
- Refund / reversal cannot be triggered by a status change out of a final state. Such a transition must be impossible; if it becomes possible through a bug, the refund path must still refuse.

## State machine — required rules

Allowed transitions (authoritative):

```
NEW        → PROCESSING
PROCESSING → APPROVED | FAILED | CANCELED | ERROR
ERROR      → PROCESSING                    (retry)
APPROVED   → REVERSED | FINAL
CANCELED   → FINAL
REVERSED   → FINAL
FAILED     → FINAL                          (no further mutation)
```

Rules:
- Every state transition must be guarded: current state + event → next state. No ad-hoc `UPDATE status=...`.
- Final states are **terminal and immutable** — `APPROVED → REVERSED → FINAL`, `FAILED → FINAL`, `CANCELED → FINAL`, `REVERSED → FINAL`. Once a row is `FINAL` (or in `FAILED`), no further status change is allowed from any source — callback, webhook, retry, operator.
- Final-state immutability must be enforced at the **database level** (CHECK constraint, trigger, or WHERE-clause guard), not only in app code. App-level guards fail when another service / job / hotfix bypasses the handler.
- Transitions to `APPROVED` / `FAILED` must be driven by an authoritative source (see `review-error-classification`). Infra errors cannot cause `FAILED`.
- A repeated request or late callback for an already-final transaction must **not** overwrite status. Example: tx id=1 is `APPROVED`; a later webhook arrives with `FAILED` for id=1 — **ack the callback so the sender stops retrying, log, alert, do not update the row.**

## Final-state immutability and late / stale callbacks

A late callback overwriting a final status is a class of bug that has directly caused incorrect refunds. This section is the bar for every status-update code path.

### Rules

- [ ] Every status UPDATE is written as a **guarded transition** — the WHERE clause includes the expected current state:
  ```sql
  UPDATE transactions
     SET status = 'APPROVED', completed_at = now(), last_updated_at = now()
   WHERE id = $1
     AND status = 'PROCESSING';
  ```
  If the updated-row-count is 0 → the row was not in the expected state. Do **not** fall back to a blind update. Log, alert, route to reconciliation.
- [ ] A **DB-level** defense exists in addition to the app-level guard: a trigger or CHECK constraint that rejects any UPDATE when the current status is `FINAL`, `FAILED`, `CANCELED`, or `REVERSED`. Belt and suspenders.
- [ ] Late / stale / conflicting callbacks are treated as **reconciliation events**: ack the sender, persist the raw payload with a `conflict` flag, alert on-call, route to manual review. They do not touch the transaction row.
- [ ] Callbacks / webhooks carry a **timestamp** (or monotonic sequence) from the sender. Updates older than the row's `last_updated_at` (or the transition that moved the row to its current state) are rejected as stale.
- [ ] Callback handlers are **idempotent**: a replay of the same callback does nothing beyond returning the same ack.
- [ ] **Refund / reversal is never triggered as a side effect of a status change out of a final state** — such a transition is impossible by the DB guard, and even if it leaks through, the refund handler must re-check the source of truth before paying out. (Cross-reference: `review-external-api` → Refund and reversal guards.)

### Example — wrong vs right

**Wrong** — blind update, no current-state guard, no DB defense, refund triggered on overwrite:

```go
func handleCallback(tx Callback) error {
    _, err := db.Exec(`UPDATE transactions SET status = $1 WHERE id = $2`, tx.Status, tx.ID) // ❌ overwrites APPROVED → FAILED
    if err != nil { return err }
    if tx.Status == "FAILED" {
        return refund.Issue(tx.ID)  // ❌ money returned even though tx was previously APPROVED
    }
    return nil
}
```

**Right** — guarded transition, stale-check, final-state respected, refund gated on actual prior state:

```go
func handleCallback(cb Callback) error {
    res, err := db.Exec(`
        UPDATE transactions
           SET status = $1, last_updated_at = now()
         WHERE id = $2
           AND status = 'PROCESSING'
           AND last_updated_at <= $3`,
        cb.Status, cb.ID, cb.SenderTimestamp)
    if err != nil { return err }
    if n, _ := res.RowsAffected(); n == 0 {
        // Row was not PROCESSING, OR callback is stale.
        // Do not touch the row; route to reconciliation.
        reconcile.Enqueue(cb)
        alerts.Raise("callback_conflict", cb.ID, cb)
        return nil // ack the sender so it stops retrying
    }
    // Refund is NOT triggered here — refund lives in a separate handler
    // gated on a status-check against the authoritative source.
    return nil
}
```

## Persist before side effects

**Rule: before performing any operation with an external side effect, the transaction must be durably saved in the database.** Local persistence is the ticket that proves the operation was attempted. Without it you cannot retry, reconcile, refund, audit, or even tell the user what happened.

A "side effect" is anything the system cannot undo just by rolling back its own DB transaction: an outbound HTTP call to a provider, a message published to a broker, an SMS / email / push notification, a file write, a ledger post to another service, a balance increment in a separate store, a webhook callback, an external-ID reservation.

### Why this exists
If you call the provider first and then write to the DB, any of these cheap things loses money or integrity:
- DB is briefly unavailable between the call and the save → provider charged the customer; local record does not exist → reconciliation miss → silent loss.
- Application crashes between the call and the save → same outcome.
- DB rollback happens after the external call → provider thinks operation succeeded; local state denies it.
- The broker publish goes out before commit and the outer DB transaction later rolls back → consumers act on an event that never happened.

### Rules

- [ ] The transaction row is **INSERTed and committed** (or written into an Outbox in the same local transaction) **before** any outbound call, publish, or notification.
- [ ] Unique identifiers the external side effect depends on (idempotency key, external reference, request ID) are generated and persisted with the row **before** the side effect, not after.
- [ ] The DB write and the side effect are **never** performed such that the side effect can succeed while the DB write is rolled back or fails silently.
- [ ] When a broker publish is needed atomically with a DB write → use the **Outbox pattern** (see `review-async-transactional`). Never "commit DB, then publish" as a single if-statement — the process can die between the two steps.
- [ ] Side effects that do not need strict atomicity (notifications, analytics, audit export) run in a **background job** driven by the persisted row, not inline in the request path. On worker crash, the row is still there; the job is retried.
- [ ] The persisted row always carries enough state (status, external refs, amounts, actor, timestamps, raw request body if relevant) to fully reconstruct what the side effect was supposed to do.
- [ ] On restart / crash recovery, a reconciliation / status-check job finds `PROCESSING` rows whose side effect status is unknown and resolves each one against the authoritative source.

### Red flags

- [ ] External HTTP / gRPC call executed before `INSERT` / `db.Save` of the transaction row
- [ ] Broker publish (`channel.Publish`, `producer.Send`, `nats.Publish`) executed before the DB commit — and **not** via an outbox
- [ ] Notification / SMS / email / webhook fired before the transaction row is committed
- [ ] Balance increment / decrement written to a separate store (cache, Redis, counter service) before (or without) the authoritative DB write
- [ ] Idempotency key / external reference generated inside the side-effect call, not beforehand and persisted
- [ ] Non-transactional work (analytics, audit, notifications) executed inline after the side effect but before the response, where a DB failure leaves orphaned effects
- [ ] No reconciliation / status-check job to resolve `PROCESSING` rows after a crash — the system depends on the happy path always completing

### Example — wrong vs right

**Wrong** — provider charged; local row never saved:

```go
func pay(req Request) error {
    resp, err := provider.Charge(req.Amount, req.CardRef) // ❌ side effect first
    if err != nil { return err }
    tx := Transaction{ID: req.ID, Status: "APPROVED", ProviderRef: resp.ID, Amount: req.Amount}
    return db.Save(&tx)                                    // ❌ if this fails, we have no record of the charge
}
```

**Right** — row first (with idempotency key), then side effect, then update from the real result:

```go
func pay(req Request) error {
    tx := Transaction{
        ID:             req.ID,
        Status:         "PROCESSING",
        IdempotencyKey: newIdempotencyKey(req.ID),
        Amount:         req.Amount,
        CreatedAt:      time.Now().UTC(),
    }
    if err := db.Save(&tx); err != nil { return err } // persisted before any side effect

    resp, err := provider.Charge(srvCtx, req.Amount, req.CardRef, tx.IdempotencyKey)
    // Error branches classified per review-error-classification.
    // Status update uses a guarded transition (see Final-state immutability above).
    return reconcile.ApplyProviderResponse(tx.ID, resp, err)
}
```

## Timestamps — required semantics

| Field | When set | Mutable? | Notes |
|---|---|---|---|
| `CREATED_AT` | On INSERT (NEW) | No | UTC, server-assigned |
| `LAST_UPDATED_AT` | On every status / field change | Yes | Mainly during `PROCESSING`; can record last attempt |
| `COMPLETED_AT` | On entry to a final state | No | Only set once |

- All timestamps stored in **UTC**. Never local time.
- Millisecond precision (or better) to support accurate sequencing.
- Clear retention policy: how long transaction rows + timestamps are kept.
- Once set, immutable fields stay immutable — enforce at the DB level where possible.

## Monetary and immutable fields

- Store money as **integers in minor units** (cents, kopecks, tiyin). Never float. Never string-without-precision-tag.
- Immutable fields (once set, must not change): `id`, `created_at`, `amount`, `currency`, `user_id` / `sender`, `receiver`, `transaction_type`, `external_ref`, `gate_ref`.
- Enforce immutability via triggers or application-level guards; prefer DB-level.

## Exchange rates and fees

- [ ] Rate and fee shown to user **before** confirmation
- [ ] Rate has an **expiration timestamp**; expired rates cannot be used to complete
- [ ] Volatility guard — reject or alert if rate or fee-to-amount ratio moves beyond threshold between quote and confirm
- [ ] Fallback secondary rate source if primary is unavailable; never silently substitute
- [ ] Rate source recorded with each transaction (audit trail)

## Keys, constraints, storage

- [ ] Primary key on every transaction table
- [ ] Foreign keys to user / account / wallet tables
- [ ] **Unique** constraints on business-unique fields: `(client_id, client_request_id)`, `external_ref`, `gate_ref`
- [ ] For partitioned tables: uniqueness enforced across partitions (global unique index or design that prevents cross-partition duplicates)
- [ ] Transaction data lives in a **single table** — do not split amount/status/user across multiple tables requiring joins for correctness

## Concurrency, transactions, connections

- [ ] ACID transaction wrapping any multi-row change (`BEGIN ... COMMIT/ROLLBACK`)
- [ ] All transactional writes use the **master** DB connection, not a replica
- [ ] Row persisted in DB **before** any outbound call, broker publish, notification, or any other side effect (see **Persist before side effects** section above — this is its own required review step)
- [ ] Cancel / rollback logic in a **separate handler**, not mixed into the main request path
- [ ] Both commit and rollback paths tested and instrumented
- [ ] DB and context timeouts configured; inbound HTTP request context **not** used for DB operations
- [ ] Connections closed / returned to pool on every path (including panics / exceptions)

## Authorization and isolation

- [ ] User A cannot read or mutate user B's transactions — object-level permission check on every query and mutation
- [ ] Admin / back-office mutations go through a separate code path with audit logging

## PR review checklist

1. Read the migration / schema diff. Confirm PK, FKs, unique constraints, NOT NULL on required fields, money stored as integer.
2. Grep the diff for `UPDATE`, `update(`, ORM `.save(`, `.update(`, or any write against the transaction table. For each: does the WHERE clause (or ORM guard) include the **expected current state**? A blind `WHERE id = ?` is a block-the-PR finding.
3. For every status-update call site: what does the code do when 0 rows are affected? Silent continuation is wrong — correct behavior is log + alert + route to reconciliation, never a fallback to a blind update.
4. Check for a constraint/trigger preventing mutation of final-state rows. If missing, flag it as a required migration.
5. Find every webhook / callback / async-event handler that can write the transaction status. For each: confirm guarded transition, stale-timestamp check, idempotent replay, and that no refund is triggered as a side effect of a status overwrite.
6. **Persist-before-side-effects:** grep the diff for outbound HTTP / gRPC calls, `publish` / `send` to brokers, SMS / email / push notifications, cache or counter writes, and webhook calls. For each: is a transaction row committed (or an outbox row written) in the DB **before** the call? If the sequence is `side effect → save`, or `save → publish` without an outbox → block.
4. Grep for `float`, `double`, `decimal` on amount/fee fields. Flag anything not integer minor units or properly-typed decimal with known precision.
5. Check timestamp columns: UTC? Set-once fields guarded?
6. For exchange-rate code: find the expiration check, the volatility guard, the fallback source.
7. Confirm the DB connection used for writes is the master. Grep for replica/read-replica usage inside transactional paths.
8. Confirm rollback handler is separate from the main handler.
9. Check object-level authorization on every query / mutation touching another user's resources.
10. Cross-check with `review-error-classification` for error-branch state transitions, and `review-sync-transactional` / `review-async-transactional` for context and timeout rules.
