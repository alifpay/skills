---
name: review-async-transactional
description: Review asynchronous transactional services built on message brokers, queues, or streaming (RabbitMQ, Kafka, NATS JetStream, SQS, Pub/Sub). Checks broker retention and durability, delivery-reliability level, duplicate-message handling (retry without ACK, recovery after crash, stale months-old messages), idempotent consumers, Outbox/Inbox patterns, transactional boundaries across DB + broker, request freshness/valid-until, persistence flags, Publisher Confirms, Consumer Acknowledgements, Dead Letter Exchanges/Queues, reconnect strategy, and poison-message handling. Trigger when reviewing producers, consumers, event handlers, saga/orchestrator code, or any async path that moves money or mutates transactional state.
---

# Asynchronous Transactional Service Review

## Why this skill exists
Message brokers make systems scalable but they break naive transactional thinking. Messages can be delivered twice, delivered months late, lost on broker crash, or committed to the broker while the DB rollback wipes the logical source. Every consumer must assume duplicate and out-of-order delivery. Every producer must assume the broker can fail between the DB commit and the publish.

## Red flags to catch during review

- [ ] Publish-to-broker inside the same function as the DB commit, with no Outbox table — **if the broker call fails after DB commit, the message is lost forever**
- [ ] DB write happens **after** the publish — on DB failure a message exists for a transaction that does not
- [ ] Consumer has no idempotency check; processes the same message twice and double-spends
- [ ] No unique-per-client request ID persisted in the Inbox before processing
- [ ] Consumer processes a message with no freshness check — a message from months ago is executed as if fresh
- [ ] Consumer writes the transaction status with a blind `UPDATE ... WHERE id = ?` (no current-state guard) — **a late / replayed / out-of-order message can overwrite a final status, and if a refund path is wired to the status change, money is returned on a successful transaction.** Updates from consumers must be guarded transitions with stale-timestamp rejection (see `review-transaction-model` → Final-state immutability).
- [ ] Consumer writing to a final-state row does not route to reconciliation — silently drops or silently overwrites
- [ ] Retention / TTL not set on durable queues; messages live forever and resurface after storage rot
- [ ] Messages not marked persistent / durable; broker restart loses in-flight work
- [ ] Queue not declared durable; broker restart loses the queue
- [ ] Auto-ack enabled on consumer; crash mid-processing drops the message
- [ ] Manual ack sent **before** the DB commit on the consumer side
- [ ] No Publisher Confirms (RabbitMQ) / no `acks=all` (Kafka) — producer gets "ok" before broker has persisted
- [ ] No Dead Letter Exchange / Queue for messages that repeatedly fail
- [ ] Poison-message loop — message fails, gets redelivered forever, blocks the queue
- [ ] Reconnect logic absent or incorrect — a connection drop kills the consumer and it does not restart
- [ ] DB transaction wraps the broker publish call — broker latency holds DB locks; broker failure rolls back the DB even though the message may have been sent
- [ ] Saga / orchestrator without persisted step state — a crash mid-saga loses progress
- [ ] Consumer processes messages concurrently on the same aggregate without ordering / partition guarantees

## Message / event schema versioning

Messages and events are an API. Breaking a message schema in place breaks every consumer at once and is usually worse than breaking an HTTP API (consumers are not always under your control, redelivery amplifies the damage, old messages may still be in the queue).

### Rules

- [ ] Every published message / event carries an explicit **schema version** (e.g. `v1`) — in the routing key, topic name, message header, or payload envelope.
- [ ] **Breaking changes to a message schema are forbidden in the same version.** A breaking change creates a new version (new topic, new routing key, new event type), and old + new run in parallel until every consumer migrates and the old queue drains.
- [ ] Breaking for messages is the same list as for HTTP (remove field, rename, change type, change enum semantics, change required/optional, change authentication/signing). Additionally: changing ordering / partitioning guarantees, changing delivery semantics (at-least-once → exactly-once), changing retention.
- [ ] Consumers are written to **ignore unknown fields** (forward-compat) — a new optional field in the producer must not break a consumer on an older version.
- [ ] When retiring an old version: confirm from metrics that the old queue / topic has been empty for at least the retention window before deleting.

### Red flags

- [ ] Producer changes a field type / removes a field / renames a field on an existing topic / routing key / event type
- [ ] New message type published without a version marker
- [ ] Consumer crashes on an unknown field (forward-compat violation)
- [ ] Old message-schema version retired while messages may still be in flight or in DLQ
- [ ] Cross-reference with `review-sync-transactional` → API versioning discipline for the full breaking-change list

## What the code SHOULD do

1. Use the **Outbox pattern** on producers: the DB write and an `outbox` row are committed in a single local transaction. A separate relay process publishes from the outbox with retry.
2. Use the **Inbox pattern** on consumers: persist the message's unique ID before processing; reject on duplicate; ack only after the DB commit.
3. Require a unique client-provided request ID on every transactional message; enforce uniqueness at the DB level.
4. Require a `transaction_date` / `valid_until` field; reject stale messages with a clear "expired" reason (do not silently drop).
5. Configure durable queues, persistent messages, appropriate retention / TTL, and a Dead Letter target.
6. Enable Publisher Confirms (RabbitMQ) / `acks=all` + idempotent producer (Kafka).
7. Consumer uses **manual ack after commit**: process → commit DB → ack broker. If crash between commit and ack, redelivery is harmless because the Inbox rejects the duplicate.
8. Never wrap a broker call inside a DB transaction. Use the Outbox instead.
9. Define retry/backoff on the consumer; after N failures, route to DLQ + alert.
10. Implement reconnect with exponential backoff; monitor consumer lag + DLQ depth.
11. For sagas/orchestrators: persist each step's input, output, and state; make every step idempotent and resumable.
12. Partition / route messages so that operations on the same aggregate (e.g. same account) are processed in order.

## PR review checklist

1. On the producer side: find the publish call. Is it inside a DB transaction? If yes → flag (use Outbox). Is the DB write committed before publish? If no → flag.
2. On the consumer side: find the ack call. Is it before or after the DB commit? Must be after.
3. Grep for `autoAck`, `enable.auto.commit=true`, `ackMode=auto` — almost always wrong for transactional consumers.
4. Look for the duplicate check. Is there an Inbox table / seen-messages cache? Is the message's unique ID persisted with a uniqueness constraint?
5. Look for the freshness check. Is `valid_until` / transaction_date validated?
6. Check queue / topic configuration: durable, persistent, retention, DLQ.
7. Check broker client settings: Publisher Confirms on; producer idempotence on; acks=all; retries configured.
8. Check reconnect and error-backoff logic. Is there a DLQ route after N failures?
9. For sagas: confirm step state is persisted per step and each step is idempotent.
10. Cross-check with `review-error-classification` for how infra errors are classified.
