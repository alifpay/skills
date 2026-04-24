---
name: review-database-design
description: Review backend database design decisions for read/write routing (master vs replica) and indexing. Checks that transactional writes and consistency-critical reads use the master, that analytics/reporting reads use replicas, replication-lag awareness, failover behavior, read-write split correctness, index coverage for WHERE and date-range queries (created_at, completed_at), compound indexes for multi-column filters, unique-constraint indexes (external_ref, gate_ref), index maintenance (usage monitoring, insert/update overhead, storage, low-traffic creation window), and prevention of replica-reads inside transactions. Trigger when reviewing query code, repositories, ORM configuration, migrations, read-routing logic, or performance-related PRs.
---

# Database Design Review

## Why this skill exists
Two cheap mistakes cause expensive outages: routing consistency-critical reads to a replica (stale data → wrong decisions) and missing indexes on hot query paths (query latency → connection-pool exhaustion → cascading failure). Both look fine in code review unless someone is looking for them specifically.

## Master vs. replica routing

### Must use the master
- [ ] All writes
- [ ] **All reads inside a transaction** — replica lag can cause the transaction to read stale data and make decisions that contradict what it will write
- [ ] Consistency-critical reads: authentication, balance checks before debit, fraud/limit checks, idempotency lookups, final-state reads immediately after write
- [ ] Any read whose result feeds a subsequent write

### Safe on replicas
- [ ] Analytics, reporting, dashboards
- [ ] Read-heavy list/search endpoints where seconds of lag are acceptable
- [ ] Background aggregation jobs tolerant of lag

### Red flags
- [ ] ORM / repository defaulting every read to a replica without a per-query override
- [ ] Balance / limit / idempotency read against a replica
- [ ] Transaction handler opening one connection to master for writes and another to replica for reads within the same logical operation
- [ ] No failover: replica unavailable → queries hang instead of falling back to master
- [ ] Replication-lag monitoring absent; no alert threshold

### What the code should do
1. Default to master for any operation under a transaction boundary.
2. Mark replica-safe reads **explicitly** (e.g. `WithReplica()`, `readOnly: true`) rather than defaulting to replica.
3. Monitor replication lag; alert when it exceeds threshold.
4. Failover logic: if replica returns error / times out, retry on master.
5. Load-balance across replicas for analytic reads to spread load.

## Indexes

### Indexes that should exist
- [ ] WHERE-clause columns used by production queries: `user_id`, `external_ref`, `gate_ref`, `status`
- [ ] Date-range columns: `created_at`, `completed_at` (for reporting + status-check jobs)
- [ ] **Compound indexes** where queries filter on multiple columns together: `(status, created_at)`, `(user_id, created_at)`, `(user_id, status)` — order matters (most-selective first or matching the query predicate left-to-right)
- [ ] Unique constraints on business-unique columns (`external_ref`, `client_request_id`) — the constraint's underlying index doubles as a lookup index

### Red flags
- [ ] New query added with a WHERE / ORDER BY column that has no supporting index
- [ ] `SELECT ... WHERE status='PROCESSING'` with no index on `status` (or `(status, created_at)`)
- [ ] Migration adds an index on a large table with no online/concurrent option and no maintenance-window note
- [ ] Index added but never referenced by a query — dead index costs write overhead for nothing
- [ ] Redundant indexes (e.g. separate `(a)` and `(a, b)` — the `(a, b)` index already covers `(a)` lookups)
- [ ] Compound index column order does not match the query pattern

### What the code should do
1. Every new query path is reviewed for an index that supports it — run `EXPLAIN` on production-like data.
2. Unique constraints added for every business-unique field; rely on their indexes instead of duplicating.
3. Index creation migrations use `CREATE INDEX CONCURRENTLY` (Postgres) or equivalent online option on large tables.
4. Schedule index creation in the lowest-traffic window; note it in the migration.
5. Monitor index usage (`pg_stat_user_indexes` or equivalent); drop unused indexes in periodic cleanup.
6. Review the write-amplification impact — every index adds cost to INSERT/UPDATE.
7. Budget for storage growth when adding wide indexes.

## PR review checklist

1. Grep the diff for every new or changed query. For each: which connection (master/replica) is it using? Is the choice correct?
2. For any query inside a transaction: confirm master.
3. For balance / limit / idempotency / auth reads: confirm master.
4. For every new WHERE / JOIN / ORDER BY / GROUP BY column in a production query: is there a supporting index? Run `EXPLAIN` against realistic data.
5. For every new index in a migration: is it concurrent/online? Is the maintenance window called out? Is it actually used by a query?
6. Look for redundant or shadow indexes.
7. Confirm unique constraints exist on all business-unique fields (cross-check with `review-transaction-model`).
8. Check replication-lag monitoring and failover behavior if replica routing is added or changed.
