---
name: review-api-testing
description: Review the test coverage and QA plan for transactional/financial APIs. Checks functional coverage (create/process flows, status updates, transaction types, limits, fees, transfers, partial failures), security coverage (encryption in transit and at rest, authN/authZ, injection, rate limiting, input validation, fraud detection), load coverage (peak concurrency, recovery after stress), integration coverage (external APIs, database consistency), logging (detail + no sensitive data), regression coverage, and QA skill/tooling requirements (Postman, Swagger, SQL, JMeter, Grafana/Kibana, Git). Trigger when reviewing test suites, QA plans, test-regulation documents, CI changes that affect test scope, or new transactional endpoints without matching tests.
---

# Transactional API Testing Review

## Why this skill exists
Money-moving code that ships without matching test coverage is where incidents breed. This skill exists to make "tests exist" visible during PR review: not just unit tests, but functional coverage for all transaction types, security coverage for known attack surfaces, load coverage for peak conditions, and integration coverage for external dependencies.

## 1. Functional coverage

Required scenarios per transactional endpoint:
- [ ] Happy-path create + process + final status
- [ ] Every status transition reachable and asserted (NEW → PROCESSING → APPROVED/FAILED/CANCELED/ERROR → FINAL)
- [ ] Status notifications / webhooks fire on transition
- [ ] Every **transaction type** the endpoint supports: inbound, outbound, P2P, bill pay, refund, reverse
- [ ] Every **currency** supported, including conversion at current rate
- [ ] Limits: max/min amount, per-period count — both below and above the threshold
- [ ] Fees: correct calculation, display before confirmation, actual deduction matches display
- [ ] Transfers: both sender and receiver balances updated; rollback on any partial failure
- [ ] Partial failures / edge cases: connection drop, server timeout, client cancel, insufficient funds, expired card, invalid data
- [ ] Duplicate-submission test: send the same idempotent request twice immediately, and again after a delay — assert the second is a no-op returning the first result

## 2. Security coverage

- [ ] Transport: TLS enforced for card data, PII, and all transaction traffic; HTTP/plaintext blocked
- [ ] Storage: sensitive transaction data encrypted at rest; no plaintext PAN/CVV/PIN anywhere
- [ ] AuthN on every transactional endpoint; strong factor required (OTP, biometric, hash-token) for money-moving actions
- [ ] AuthZ object-level: user A cannot read / modify / create against user B's resources — tested explicitly
- [ ] Unauthorized-action tests: attempt to modify / add transaction data without permissions, expect rejection + audit log
- [ ] Auth-bypass tests: missing / expired / tampered token → reject
- [ ] Input validation tests: valid + invalid + boundary values + missing params + wrong types + special chars → appropriate error messages
- [ ] Injection tests: SQL, XML, NoSQL, command — automated where possible
- [ ] Rate-limiter tests: exceed the limit and confirm blocking + reset
- [ ] Long / oversized payload handling — no crash, clear error
- [ ] Currency / card-type / locale matrix tested (not only one case)
- [ ] Fraud detection tests: abnormally large amount, unusual geography, unusual time-of-day, high frequency, mismatched data → blocked + alert
- [ ] No sensitive data in logs (grep-style test in CI)

## 3. Load coverage

- [ ] Peak concurrency: large number of simultaneous transactions processed correctly (no duplicates, no stuck states)
- [ ] Response-time SLO held under user-growth scenarios
- [ ] Recovery: after sustained peak, system returns to baseline without stuck transactions
- [ ] Tool: JMeter (or equivalent) scripts checked into the repo

## 4. Integration coverage

- [ ] External providers: success, each documented error code, timeout, unknown response — each mapped to the right local classification (cross-check `review-external-api`)
- [ ] Provider-error logging verified (correct level, correlation ID, no sensitive data)
- [ ] Database: transaction records correctly inserted, updated, read; consistency across tables asserted
- [ ] Cross-system consistency: reconciliation check between local DB and provider / other services

## 5. Logging

- [ ] Each transaction event produces a log line with: who, what, when, correlation ID, outcome
- [ ] No PAN / CVV / PIN / password / full token in logs — verified by lint or test
- [ ] Log level appropriate: errors are errors, expected rejections are info/warn

## 6. Regression coverage

- [ ] Any change to a transactional endpoint re-runs the full functional + security + integration suite
- [ ] Change to shared code (auth, ledger, broker client) triggers regression on every dependent endpoint

## QA tooling baseline (must be available)

- API testing: Postman, Swagger
- Database validation: SQL skills
- Load testing: Apache JMeter
- Observability: Grafana, Kibana (or team equivalent)
- Version control: Git
- Programming: at least one language matching the service stack

## PR review checklist

1. New transactional endpoint in the diff? Confirm a functional test exists for happy path + every error classification + duplicate submission.
2. Change to the state machine? Confirm every transition has a test.
3. New external-provider call? Confirm tests exist for each response classification (Fatal / Retryable / Pending / Unknown).
4. Any log line added with user / card / token data? Fail review unless masked.
5. Any new authorization path? Confirm an A-cannot-access-B test exists.
6. Performance-sensitive change? Confirm load script updated or noted as follow-up.
7. For a purely test-coverage PR, confirm it asserts actual behavior, not just "200 OK".
