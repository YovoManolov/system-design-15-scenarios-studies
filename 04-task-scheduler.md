# 04. Distributed Task Scheduler & Orchestrator

> Run millions of delayed and recurring jobs at the right time, at least once, without two nodes running the same job destructively.

## 1. Intent
- "Correct": jobs fire near their time (define tolerance), are not lost, and duplicate execution is bounded and safe.
- Clarify: one-off vs cron? Precision (seconds vs minutes)? Job runtime (ms vs hours)? Exactly-once needed, or at-least-once with idempotent jobs? Priorities, retries, dependencies (DAG)?

## 2. Numbers
- 100M scheduled jobs stored, 10k firing/s at peak (top of every minute is a spike, not an average).
- Job metadata ~500 B: 100M is ~50 GB. Store in a partitioned DB, not all in RAM.

## 3. Fault lines
1. **Who owns which job?** Two schedulers must not both dispatch the same job.
2. **Zombie owner**: a node pauses (GC, network partition), loses its lease, wakes up and *still thinks it owns the job*.
3. **Polling the DB for due jobs** at scale is expensive.
4. **Top-of-minute spike.**
5. **Worker dies mid-job.**

## 4. Design
**Storage**: `jobs(id, partition, next_run_at, payload, status, attempt, version)`, partitioned/indexed by `(partition, next_run_at)`.

**Ownership by partition leases**
- Jobs hash into P partitions (e.g. 1024). Each scheduler node holds **leases** on a set of partitions (via etcd/ZooKeeper/DB row with expiry). Rebalance on node join/leave.

**Timing wheel for near-term jobs**
- Each scheduler loads jobs due in the next few minutes from the DB into an **in-memory hierarchical timing wheel** (O(1) insert/expire, used in Kafka's purgatory and Netty's `HashedWheelTimer`). The DB is touched in batches, not per tick.

**Dispatch**
- When a job fires, the scheduler enqueues it to a work queue (Kafka/SQS/etc.) and marks it `DISPATCHED`. Workers pull, execute, ack. **Visibility timeout / heartbeat**: if no ack, the job is redelivered.

**Fencing tokens (the key idea)**
- Every lease grant increments a monotonic **epoch/fencing token**. Every write to shared state (job status, dispatch) includes the token; the store **rejects writes with a token lower than the highest seen**.
- This defends against the zombie owner: even if it wakes up believing it holds the lease, its writes are rejected. A lease alone is *not* safe without fencing (Kleppmann's argument).

**Retries**: exponential backoff with jitter, max attempts, dead-letter queue. Misfire policy: run late jobs immediately, skip, or coalesce, chosen per job type.

## 5. Stress test
| Failure | What happens | Mitigation |
|---|---|---|
| Scheduler node dies | its partitions unowned | lease expires, another node takes over, reloads from DB |
| GC pause / partition, lease expires | two owners | fencing token rejects the stale one |
| Worker dies mid-job | job stuck | heartbeat + visibility timeout, redeliver (at-least-once) |
| Job runs twice | double side effect | job idempotency key; design jobs as idempotent |
| Top-of-minute spike | queue and workers saturated | spread cron with jitter, priority lanes, autoscale workers |
| Clock skew between nodes | early/late firing | use DB/lease-service time for ordering; tolerate skew in SLA |

## 6. Who pays
| Decision | Cost moves to |
|---|---|
| At-least-once | Job authors must write idempotent jobs |
| Short leases | Faster failover, more lease churn and false takeovers |
| In-memory wheel | Reload cost on failover; bounded precision |
| Fencing everywhere | Every storage layer must check tokens |

## 7. Drill questions
1. Explain why a lease without a fencing token is unsafe. Give the exact timeline.
2. How do you avoid scanning a 100M-row table every second?
3. What's exactly-once here, and why do you settle for at-least-once?
4. 1M jobs are all scheduled for 09:00:00. What happens?
5. How does rebalancing work when a node joins?

## 8. Practice notes (Java / Spring)
- Quartz (JDBC cluster mode), ShedLock, db-scheduler (Kagkarlsson), Temporal for orchestration. Netty `HashedWheelTimer` is a good reference for wheels.

## 9. Further reading
- Kleppmann "How to do distributed locking"; Varghese & Lauck "Hashed and Hierarchical Timing Wheels"; Temporal architecture docs.
