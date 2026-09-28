# 14. Distributed URL Shortener & Analytics

> Generate short, unique, non-guessable codes at scale; redirect in milliseconds; count clicks without slowing the redirect.

## 1. Intent
- Unique, short, fast to resolve, hard to enumerate. Analytics must never slow down redirects.
- Clarify: custom aliases? Expiry? Is guessability a concern (private links)? Analytics detail (geo, device, unique visitors)? Abuse/malware handling?

## 2. Numbers
- 100M new URLs/day = ~1.2k writes/s. Read:write ~100:1 = **~120k redirects/s**, more at peak.
- Code length: base62 (`0-9a-zA-Z`), 7 chars = 62^7 = ~3.5 trillion codes. At 100M/day that's ~96 years.
- Storage: 100M/day x 365 x 5 yr = ~180B rows x ~200 B = ~36 TB. Key-value store, sharded.
- Click events: 120k/s x ~100 B = ~12 MB/s of analytics stream.

## 3. Fault lines
1. **Unique code generation without a coordination bottleneck.**
2. **Hash-and-truncate collides** (birthday problem) and needs a DB check per write.
3. **Sequential IDs are enumerable** (competitors can scrape all links).
4. **Read path** must be cache-first and tiny.
5. **Synchronous analytics** would multiply redirect latency.

## 4. Design
**Code generation options**
- *Hash (MD5/SHA) of URL, take 7 chars*: collisions, need retry loop and lookup. Same URL from different users shares a code, which may not be wanted.
- *Random 7 chars*: collisions need check-and-retry (cheap at low fill, worse as it fills).
- *Counter + Base62*: no collisions, but sequential and guessable.
- **Counter + Feistel + Base62 (recommended)**: a **Feistel network** is a keyed permutation over a fixed-size integer domain (e.g. 42 bits): it is **bijective**, so every counter value maps to a unique output. Result: non-sequential-looking codes, zero collisions, no DB lookup to check. Then Base62-encode the permuted number.

**Key Generation Service (KGS) and block leases**
- A central counter (DB row, ZooKeeper, or DynamoDB atomic counter) hands out **ranges**: an app node leases block `[N, N+100k)` and generates IDs locally from memory. Coordination happens once per 100k URLs, not per request.
- If a node crashes, its unused range is lost: harmless (space is huge). Optional: multiple counters/regions with disjoint ranges.

**Data model**: KV store (DynamoDB/Cassandra/Redis+DB): `code -> {long_url, owner, created, expires}`. Primary key lookup only.

**Read path**
- CDN/edge cache -> app-level cache (Redis) -> DB. 95%+ hit ratio as traffic is heavy-tailed.
- **301 vs 302**: 301 (permanent) lets browsers/CDNs cache, cutting load but **hiding later clicks** from analytics and preventing edits/expiry; 302 (temporary) keeps every hit visible. Choose by whether analytics/revocation matter.

**Analytics off the hot path**
- Redirect handler emits a click event **asynchronously** (fire-and-forget to Kafka) with code, timestamp, geo (from IP), UA, referrer. Stream processing aggregates into an OLAP store (ClickHouse/Druid/BigQuery). Dashboards read aggregates. Losing a few events on failure is acceptable.

**Abuse**: check destination against malware/phishing lists at creation, rate-limit creators, allow takedown (which requires 302 or cache invalidation).

## 5. Stress test
| Failure | What happens | Mitigation |
|---|---|---|
| KGS/counter down | can't create new links | nodes have pre-leased blocks; multiple counters; reads unaffected |
| Node crash with unused range | codes skipped | acceptable waste |
| Cache stampede on a viral link | DB hot key | cache with request coalescing, TTL jitter, CDN |
| Custom alias collision | overwrite | conditional put (`IF NOT EXISTS`), return 409 |
| Analytics pipeline lag | stale dashboards | redirects unaffected; catch-up from Kafka |
| Link to malware | reputation damage | scanning, reporting, quick disable + cache purge |

## 6. Who pays
| Decision | Cost moves to |
|---|---|
| Block leasing | Wasted IDs on crash (irrelevant) |
| Feistel permutation | A bit of complexity and a secret key to manage |
| 301 | Analytics accuracy and revocability |
| Async analytics | Small event loss risk, eventual dashboards |
| Long-lived cache | Slow takedown propagation |

## 7. Drill questions
1. Compare hash, random, counter, and Feistel for code generation.
2. Why is a Feistel network collision-free?
3. Why lease ID ranges instead of calling a counter per request?
4. 301 or 302, and what's the tradeoff?
5. How do you prevent analytics from adding latency, and what do you lose?

## 8. Practice notes
- Implement a 4-round Feistel over 42 bits with an HMAC-based round function; test that `f` is a permutation on a small domain by enumerating it.

## 9. Further reading
- Format-preserving encryption (FF1/FF3 NIST SP 800-38G), Bitly/TinyURL engineering posts, Snowflake ID paper/blog for alternative ID schemes.
