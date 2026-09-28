# 05. Distributed Message Queue at Hyperscale (Kafka-style)

> Durable, ordered, replayable messaging at millions of messages per second on commodity disks.

## 1. Intent
- Decouple producers and consumers, absorb bursts, allow replay.
- Clarify: ordering scope (per key/partition is realistic; global is not scalable)? Delivery semantics? Retention (time/size/compacted)? Latency vs durability priority?

## 2. Numbers
- 1M msg/s x 1 KB = 1 GB/s ingress. With replication factor 3, ~3 GB/s of disk writes cluster-wide.
- One broker sustaining ~100-200 MB/s: 20-30 brokers just for this. Reads (multiple consumer groups) multiply egress.
- Retention 7 days x 1 GB/s = ~600 TB before replication.

## 3. Fault lines
1. **Random I/O kills disks**; sequential is fast.
2. **Copying data** user-space <-> kernel wastes CPU.
3. **Broker failure must not lose acknowledged data** or silently reorder it.
4. **Partition count**: parallelism ceiling vs per-partition overhead.
5. **Consumer group rebalances** stall processing.

## 4. Design
**Storage = append-only log per partition**
- Partition is split into **segments** (files). Writes append to the active segment. Sparse **offset index** and **time index** give O(log n) lookup.
- Sequential writes + reliance on the **OS page cache** (no separate app cache). Recent data is served from RAM.
- Retention deletes whole segments (cheap). **Log compaction** keeps the latest value per key.

**Throughput tricks**
- **Batching** on producer and broker, plus **compression** per batch.
- **Zero-copy** (`sendfile` / `transferTo`): page cache -> NIC without user-space copy, for consumers reading non-TLS data.

**Replication and ISR**
- Each partition has a leader and followers. Followers fetch from the leader like consumers.
- **ISR (in-sync replicas)**: followers caught up within `replica.lag.time.max.ms`. The **high watermark** is the offset replicated to all ISR; consumers only see up to it.
- `acks=all` + `min.insync.replicas=2` (with RF=3): a write is acknowledged only when at least 2 replicas have it. Tolerates one failure without data loss and refuses writes (not silently degrades) if fewer than 2 remain.
- **Leader epochs** let a new leader and old followers agree where to truncate divergent logs after failover.
- `unclean.leader.election.enable=false` prefers consistency: if no ISR member is alive, the partition is unavailable rather than electing a stale replica.

**Metadata / controller**: KRaft (Raft quorum) manages partition leadership and metadata; no ZooKeeper in modern Kafka.

**Consumers**: pull model; consumer tracks an **offset** (committed to `__consumer_offsets`). A partition is consumed by at most one consumer per group; parallelism <= partition count.

**Producer semantics**: idempotent producer (PID + sequence numbers) removes duplicates from retries per partition. Transactions extend this to atomic multi-partition writes plus offset commit (exactly-once *within Kafka*).

## 5. Stress test
| Failure | What happens | Mitigation |
|---|---|---|
| Leader dies | partition offline briefly | controller elects new leader from ISR; clients refresh metadata |
| Leader dies, ISR = only itself (`min.insync.replicas=1`) | acked data lost | `acks=all` + `min.insync.replicas=2`, unclean election off |
| Slow follower | shrinks ISR | monitor ISR shrink rate; investigate disk/network |
| Consumer crashes after processing before commit | reprocessed | at-least-once; idempotent consumers or transactional outbox |
| Rebalance storm | consumption pauses | cooperative-sticky assignor, static membership, tuned `session.timeout.ms` |
| Hot partition (skewed key) | one broker/consumer saturated | better key, key salting, or split downstream |

## 6. Who pays
| Decision | Cost moves to |
|---|---|
| `acks=all` | Produce latency |
| More partitions | Open file handles, replication overhead, longer failover and rebalance |
| Per-partition ordering only | Apps needing global order must design around it |
| Pull model | Consumers poll (latency floor), but they control backpressure |
| Compaction instead of retention | Consumers can't replay full history |

## 7. Drill questions
1. Why is a log on spinning disks faster than people expect?
2. Explain ISR, high watermark, and what `acks=all` really guarantees.
3. `min.insync.replicas=2`, RF=3, two brokers die. What happens to producers?
4. Why can't Kafka give exactly-once end-to-end to an external database, and what do you do?
5. How do you choose partition count?

## 8. Practice notes (Kafka / Spring)
- Spring Kafka: `DefaultErrorHandler` + `DeadLetterPublishingRecoverer`, `ContainerProperties.AckMode`, `KafkaTransactionManager`.
- Know these configs cold: `acks`, `enable.idempotence`, `max.in.flight.requests.per.connection`, `min.insync.replicas`, `isolation.level`, `auto.offset.reset`, `max.poll.interval.ms`.

## 9. Further reading
- Kafka docs (design section), KIP-98 (idempotence/transactions), KIP-101 (leader epoch), Jay Kreps "The Log".
