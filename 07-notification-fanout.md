# 07. Global Notification & Fan-Out Pipeline

> A single event ("price dropped") must reach millions of users exactly once each, through third-party push providers, without spamming.

## 1. Intent
- Deliver each intended notification **at most once to the user's experience**, on time, without overloading providers or annoying users.
- Clarify: latency target (seconds vs minutes)? Priority classes (OTP vs marketing)? Channels (push, email, SMS)? User preferences and quiet hours? Is the watcher set changing while we fan out?

## 2. Numbers (worked example: 10M watchers of one SKU)
- 10M sends at 10k/s = 1,000 s = **~17 minutes**. At 50k/s = ~3.5 minutes. Target throughput decides parallelism.
- APNs/FCM over **HTTP/2**: one connection multiplexes many concurrent streams (hundreds); 100 connections x ~500 in-flight = ~50k concurrent requests. You need a **pool of persistent connections**, not a connection per send.
- Device tokens: 10M rows ~ 1-2 GB. Fits fine.

## 3. Fault lines
1. **One partition, one consumer**: publishing a single event keyed by SKU puts all fan-out on one consumer.
2. **Timeout after enqueue**: provider slow or unclear; retry may double-send.
3. **Rapid successive events**: two price changes 800 ms apart => two alerts per user.
4. **Locking**: updating 10M watcher rows in the same transaction as the price write.
5. **Provider rate limits and invalid tokens.**
6. **Marketing traffic starving transactional traffic.**

## 4. Design
```
Price service --(1 event)--> Kafka topic: price-changes (few partitions, low volume)
                               |
                        Fan-out planner
              (reads watchers by SKU index, in id ranges)
                               |
             emits shard tasks: (event_id, sku, user_id_range)
                               v
              Kafka topic: notif-tasks (MANY partitions, keyed by shard)
                               |
             Fan-out workers (parallel, autoscaled)
    - dedupe check -> fatigue check -> render -> send
                               v
            Provider connectors (persistent HTTP/2 pools)
                     APNs / FCM / email / SMS
                               |
                    provider feedback (invalid token, 429)
```
**Break the hot partition**: the price event is small and cheap. The *fan-out stage* is what needs parallelism, so the planner splits the watcher set into shards (user_id ranges/buckets) and publishes many small tasks across many partitions.

**Watch table is read-only in the hot path**
- Price write updates **one row**. Watchers are read via an index on `sku_id`. Fan-out never updates all watchers; it only writes per-user "last notified version" during send, spread across shards.

**Idempotency**
- Key = `(user_id, sku_id, price_version)`. Before sending: insert-if-absent into a dedupe store (Redis `SET NX` with TTL or a DB unique key). If it exists, skip. Retries after timeout reuse the same key.
- Honest framing: this is **at-least-once with dedup**, not magic exactly-once. Providers also accept a collapse/dedupe id (APNs `apns-collapse-id`, FCM `collapse_key`) so a duplicate replaces rather than stacks.

**Coalescing rapid updates**
- Per `(user, sku)` debounce window (e.g. 5-30 s): keep only the latest version. Or at send time, check "is this version still the latest?" and drop if superseded.

**Fatigue caps**
- Per-user limits (N/day, N/hour per category), quiet hours by user time zone, priority classes that bypass caps (OTP, security). Counters in Redis with TTL.

**Priority lanes**
- Separate topics/worker pools for transactional vs promotional so a 10M blast never delays a password reset.

**Provider hygiene**
- Handle 429 with backoff and jitter; respect provider concurrency limits; remove invalid tokens on feedback (APNs 410 Unregistered, FCM `UNREGISTERED`).

## 5. Stress test
| Failure | What happens | Mitigation |
|---|---|---|
| Worker crashes mid-shard | shard partially sent | Kafka redelivery + dedupe key; user never gets two |
| Provider times out after accept | unknown delivery | retry with same dedupe/collapse id |
| Provider throttles (429) | send slows | adaptive rate, backoff, circuit breaker; queue absorbs |
| Two price ticks 800 ms apart | two alerts | coalesce per (user, sku), latest version wins |
| Price falls back up before send | stale alert | send-time version check |
| Watchers change during fan-out | someone missed/extra | snapshot watcher set at event time (versioned), or accept and document |
| Marketing blast | OTP delayed | separate priority lanes and quotas |

## 6. Who pays
| Decision | Cost moves to |
|---|---|
| Coalescing | Freshness: user may see the alert a few seconds late |
| Sharded fan-out | More moving parts, weaker global ordering |
| Dedupe store | Extra write on every send (must be fast, sized to TTL) |
| Fatigue caps | Business: some legit alerts suppressed |
| At-least-once + collapse id | Rare duplicate attempts (harmless if collapsed) |

## 7. Drill questions
1. One SKU, 10M watchers, one partition. Who consumes 10M sends?
2. Provider times out after you enqueued. Do you send twice?
3. Two price ticks 800 ms apart: how many alerts does a user get?
4. Does the price write lock 10M rows? How do you avoid it?
5. How long does the blast take and how do you cut it in half?
6. How do OTPs stay fast during a promotional blast?

## 8. Practice notes (Java / Kafka)
- Kafka consumer concurrency = partitions. Use a task topic with 64-256 partitions. Java `HttpClient` (HTTP/2) or Jetty/Netty for provider pools; `pushy` library for APNs.
- Reuse your outbox knowledge: price service writes DB row + outbox event atomically.

## 9. Further reading
- APNs/FCM docs (HTTP/2 provider API, collapse ids); Slack/Facebook engineering posts on notification systems.
