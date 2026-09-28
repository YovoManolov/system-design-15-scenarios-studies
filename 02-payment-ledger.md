# 02. Payment Gateway & Double-Entry Ledger

> Move money exactly once, in the presence of timeouts, retries, and external providers you don't control.

## 1. Intent
- "Correct" means: no double charge, no lost payment, every cent traceable, books always balance.
- Clarify: card, bank transfer, wallet? Do we hold funds (need a ledger) or just route? Multi-currency? Refunds, chargebacks? Audit/regulatory needs?

## 2. Numbers
- 5k payments/s peak, each producing ~4 ledger entries: 20k inserts/s. Easily sharded.
- Ledger is append-only: ~200 bytes x 20k/s x 86400 = ~350 GB/day. Partition by time, archive cold data.
- Latency budget dominated by the PSP call (300 ms to 2 s), not our DB.

## 3. Fault lines
1. **Unknown state after timeout**: we called the PSP, got no answer. Did the charge happen?
2. **Duplicate requests**: client retries, load balancer retries, queue redelivers.
3. **Ledger integrity**: sum of debits must equal sum of credits, always, even under concurrency.
4. **Hot accounts**: one merchant/fee account touched by every transaction.
5. **Distributed transaction across systems we don't own**: a PSP cannot join your 2PC.

## 4. Design
**Idempotency**
- Client sends `Idempotency-Key`. Table `idempotency(key, request_hash, status, response)` with unique constraint on key.
- Insert first; if it exists, return the stored response (or 409 if the request hash differs, or "in progress").
- Same key is propagated to the PSP as its own idempotency reference.

**Payment state machine**: `CREATED -> AUTHORIZING -> AUTHORIZED -> CAPTURED -> SETTLED` plus `FAILED`, `UNKNOWN`, `REFUNDED`. Transitions are compare-and-set in the DB; illegal transitions are rejected.

**Double-entry ledger**
- Immutable `journal_entry(id, txn_id, account_id, direction, amount_minor, currency, created_at)`. Never update, never delete. Corrections are new reversing entries.
- Invariant: for each `txn_id`, `SUM(debit) = SUM(credit)`. Enforced in the same DB transaction that inserts the entries.
- Money as **integer minor units** (cents), never floats. Store currency with every amount.
- Balances are derived. Keep a materialized `account_balance` updated in the same transaction (optimistic version column) for fast reads; a periodic job recomputes from journal and alerts on drift.

**Sagas, not 2PC**
- 2PC works *inside* one database/shard. Across service + PSP, use a saga: authorize, then record, then capture; each step has a compensation (void/refund).
- Use the **transactional outbox**: write state change + outbox row in one local transaction; a relay publishes to Kafka. No dual-write.

**Timeout handling**
- Never assume failure. Mark `UNKNOWN`, then **query the PSP by our reference** (or wait for the webhook), then resolve. Retry with the *same* key, never a new one.

**Reconciliation**
- Daily settlement file from PSP matched against our journal. Mismatches go to a review queue. This is the ultimate safety net.

## 5. Stress test
| Failure | What happens | Mitigation |
|---|---|---|
| PSP timeout after charge | unknown state | `UNKNOWN` + status query + webhook + reconciliation |
| Client retries same request | double charge risk | idempotency key + unique constraint |
| Crash between DB write and Kafka publish | lost event | outbox pattern |
| Webhook delivered twice/out of order | double credit | dedupe by event id, state machine rejects illegal transition |
| Hot fee account | row lock contention | shard the account into N sub-accounts, sum on read |
| Bug creates unbalanced entry | books corrupt | DB check at commit, daily balance audit |

## 6. Who pays
| Decision | Cost moves to |
|---|---|
| Immutable journal | Storage and read complexity (need derived balances) |
| Saga instead of 2PC | Complexity: you write compensations and handle intermediate states |
| Synchronous PSP call | User latency; async capture trades UX for resilience |
| Sub-account sharding | Balance reads must aggregate |

## 7. Drill questions
1. The PSP call times out. Walk me through what you do next, precisely.
2. Why can't you just use 2PC with the payment provider?
3. How do you guarantee debits equal credits under concurrent writes?
4. Client sends the same idempotency key with a different amount. What happens?
5. How do you find out your ledger is wrong before finance does?

## 8. Practice notes (Java / Spring)
- `@Transactional` boundaries around state change + outbox insert. Debezium or a polling relay for the outbox. `BigDecimal`/`long` minor units, never `double`.
- Unique index on `(idempotency_key)`; catch `DuplicateKeyException` and return the stored response.

## 9. Further reading
- Stripe "Designing robust and predictable APIs with idempotency"; Square "Books, an immutable double-entry accounting database"; Pat Helland "Life beyond distributed transactions".
