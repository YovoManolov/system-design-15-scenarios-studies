# System Design Case Studies (Staff-level drills)

15 production-style case studies for practicing senior/staff distributed-systems interviews.
Written for a Java/Spring/Kafka backend engineer, but the reasoning is stack-neutral.

The goal is not to memorize architectures. The goal is to be able to **derive the answer out loud, in order, with numbers, under interruption**.

## The answer skeleton (use it every time)

1. **Intent**: what is the real problem, and what does "correct" mean? Ask 3-5 clarifying questions.
2. **Numbers**: back-of-envelope for QPS, storage, fan-out, latency budget. State the math.
3. **Fault lines**: where will this break first? (hot key, single writer, unbounded fan-out, unknown state after timeout)
4. **Design**: the simplest architecture that survives the fault lines.
5. **Stress test**: for each component, "what if it times out / dies / duplicates / is slow?"
6. **Who pays**: every choice moves cost to someone (latency, money, complexity, freshness, ops). Name them before being asked.

## Index

| # | Case study | Core ideas |
|---|-----------|-----------|
| 01 | [Distributed rate limiter & abuse defense](case-studies/01-rate-limiter.md) | token bucket, Redis Lua, edge/Envoy |
| 02 | [Payment gateway & double-entry ledger](case-studies/02-payment-ledger.md) | immutable journal, idempotency, sagas vs 2PC |
| 03 | [Flash sales & virtual waiting room](case-studies/03-flash-sales-waiting-room.md) | hot-row contention, admission tokens, holds |
| 04 | [Distributed task scheduler](case-studies/04-task-scheduler.md) | timing wheels, leases, fencing tokens |
| 05 | [Distributed message queue](case-studies/05-message-queue.md) | append-only log, zero-copy, ISR |
| 06 | [Ride dispatch & proximity matching](case-studies/06-ride-dispatch.md) | H3 cells, ephemeral GPS, k-rings |
| 07 | [Global notification fan-out](case-studies/07-notification-fanout.md) | sharded fan-out, HTTP/2 to APNS/FCM, fatigue caps |
| 08 | [Chat & presence at scale](case-studies/08-chat-presence.md) | 10M WebSockets, epoll, lazy presence |
| 09 | [Cloud file storage & chunked sync](case-studies/09-file-sync.md) | FastCDC, content-addressable storage, S3 |
| 10 | [Ultra-low-latency matching engine](case-studies/10-matching-engine.md) | LMAX disruptor, single thread, WAL |
| 11 | [News feed fan-out & ranking](case-studies/11-news-feed.md) | push-pull hybrid, k-way merge, DLRM |
| 12 | [Video streaming & transcoding](case-studies/12-video-streaming.md) | GOP splitting, ABR ladder, CDN |
| 13 | [Search autocomplete](case-studies/13-typeahead.md) | FST, Kafka clickstream, hot swap |
| 14 | [URL shortener & analytics](case-studies/14-url-shortener.md) | Feistel, block leases, Base62 |
| 15 | [Collaborative document editing](case-studies/15-collab-editing.md) | OT, CRDT, vector clocks |

## How to practice

- Pick one file. **Read only the "Intent" and "Numbers" sections**, then set a 10-minute timer and talk through the rest out loud. Record it.
- Compare against the file. Every place you said "uhh, maybe" is your study list.
- Then read the "Drill questions" and answer each in under 60 seconds.
- Log the session in [STUDY_LOG.md](STUDY_LOG.md).

Suggested order for a Kafka/backend profile: 07, 02, 03, 04, 05, 01, 14, 13, then the rest.

## Caveats

These are study notes, not gospel. Numbers are order-of-magnitude assumptions. Verify specifics against primary sources (papers, engineering blogs, docs) before quoting them in an interview.

See [TEMPLATE.md](TEMPLATE.md) to add your own case studies.
