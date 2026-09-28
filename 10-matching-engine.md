# 10. Ultra-Low-Latency Order Matching Engine

> Match buy/sell orders deterministically in microseconds, and survive a crash without losing or reordering an order.

## 1. Intent
- Correctness: strict **price-time priority**, deterministic results, no lost/duplicated orders. Latency: microseconds at the engine, consistent tail.
- Clarify: instruments count, orders/sec per instrument, order types (limit, market, IOC, cancel), pre-trade risk, market data fan-out.

## 2. Numbers
- Hot instrument: 100k-1M orders/s. Budget ~1-10 microseconds per order at the matcher.
- A lock acquisition or a GC pause of even 1 ms is 1,000x the budget.
- Order book in memory: millions of resting orders x ~100 B = hundreds of MB. Fits in RAM and CPU cache-friendly structures.

## 3. Fault lines
1. **Contention**: locks/queues between threads dominate latency.
2. **Determinism**: replicas must produce identical results to fail over.
3. **GC and allocation pauses** (in Java especially).
4. **Durability without disk latency on the hot path.**
5. **Sequencing**: who decides the order of two simultaneous orders?

## 4. Design
**Single-threaded matching core per instrument (the key insight)**
- One thread owns the order book and processes commands sequentially: **no locks, no races**, deterministic. Throughput comes from doing very little per event, not from parallelism. Scale by **sharding instruments** across cores/machines.

**LMAX Disruptor architecture**
```
Gateway(s) -> [Sequencer] -> Ring buffer -> Journaler (WAL)
                                        -> Replicator
                                        -> Matching engine (single thread)
                                                 |
                                       Output events -> market data, executions, risk
```
- **Ring buffer**: pre-allocated, fixed-size circular array; producers claim slots via sequence numbers (CAS), consumers track their own sequence. No allocation, cache-friendly, no locks.
- Consumers (journal, replicate, match) run in parallel over the *same* input sequence; matcher only proceeds after journaling is durable (or the replicated copy acks).
- **Mechanical sympathy**: avoid false sharing (pad hot fields to cache line), keep data contiguous, avoid pointer chasing and boxing, off-heap/pre-allocated objects, pinned threads, low-GC settings.

**Event sourcing / state machine replication**
- Input = totally ordered log (**WAL**) of commands. Engine state = pure function of that log. Replicas (hot standbys) consume the same log and hold identical state; failover promotes a standby. Recovery = **snapshot + replay** since the snapshot.

**Order book structure**
- Per side: price levels in a sorted structure (array/tree/skip list), each level a FIFO queue (time priority). Hash map `order_id -> order` for O(1) cancel.

**Pre-trade risk** runs before the sequencer (or as a stage) with in-memory limits; keep it lean.

**Sequencer** gives the single global order: whoever it stamps first wins. Clear and auditable.

## 5. Stress test
| Failure | What happens | Mitigation |
|---|---|---|
| Matcher crashes | state in RAM lost | replay WAL from snapshot; or fail over to hot standby that already applied the same log |
| WAL disk slow | latency spike | fast NVMe / battery-backed, batch group commits, or synchronous replication to standby instead of fsync |
| GC pause | latency outlier | allocation-free hot path, off-heap, pre-sized pools, tuned collector (or non-GC language) |
| Standby diverges | inconsistent failover | determinism: no wall-clock/random in the engine, time comes from the log |
| Hot instrument overload | queue grows | backpressure at gateway, reject or throttle (never drop silently) |
| Duplicate order from client retry | double execution | client order id dedupe at sequencer |

## 6. Who pays
| Decision | Cost moves to |
|---|---|
| Single thread | Per-instrument throughput ceiling; must shard by instrument |
| Sync journaling | Latency; async journaling risks losing acked orders |
| Determinism | No ad-hoc features using time/randomness inside the engine |
| Low-GC engineering | Code readability and dev velocity |

## 7. Drill questions
1. Why is one thread faster than eight here?
2. What is a ring buffer and why does it beat a blocking queue?
3. How do you recover after a crash with zero lost orders?
4. What makes replicas identical?
5. Where does time priority actually come from?

## 8. Practice notes (Java)
- LMAX Disruptor library, Chronicle Queue/Aeron for journaling and messaging, Agrona for off-heap collections. Kafka is **not** on the hot path here (latency too high); it can consume the *output* for downstream systems.

## 9. Further reading
- Martin Fowler "The LMAX Architecture"; Martin Thompson "Mechanical Sympathy" blog; Aeron docs.
