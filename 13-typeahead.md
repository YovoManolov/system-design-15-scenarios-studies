# 13. Search Autocomplete & Typeahead

> Suggest completions within ~100 ms as the user types, ranked by popularity and freshness.

## 1. Intent
- Return top-K (5-10) relevant completions for a prefix, very fast, reasonably fresh.
- Clarify: personalized or global? Freshness (trending in minutes vs daily)? Languages? Blocklists/safety? Typo tolerance?

## 2. Numbers
- 10B searches/day; every keystroke can trigger a request: ~100k+ suggestion QPS, 10x with per-keystroke calls (client debounce cuts this).
- Latency budget: p99 < 100 ms end-to-end, so the server should be <10-20 ms.
- Corpus: 100M distinct popular queries x ~30 B = ~3 GB. **Fits in memory on one machine**: replicate, don't shard, at first.

## 3. Fault lines
1. **Latency**: can't hit a database per keystroke.
2. **Top-K per prefix** must be precomputed or the lookup scans all completions.
3. **Freshness vs stability**: trending queries vs computing cost.
4. **Rebuilding the index** without downtime or torn reads.
5. **Abusive/unsafe suggestions.**

## 4. Design
**Data pipeline**
- Search logs -> **Kafka clickstream** -> stream/batch aggregation (windowed counts, decayed by recency; Flink/Spark) -> per-query popularity score.
- Filter: minimum frequency (kills noise/PII), blocklist, spam, offensive terms.

**Index structure**
- Naive **trie**: each node stores top-K completions. Fast, but memory-heavy with pointers.
- **FST (finite state transducer)**: like Lucene's, compresses shared prefixes *and* suffixes into a compact automaton with weights, orders of magnitude smaller than a pointer trie, still O(prefix length) lookups. Keep top-K per node or use weighted-FST top-N search.
- Build offline every N minutes/hours into an immutable artifact (file), version-tagged, pushed to object storage.

**Serving with hot memory swap**
- Serving nodes load new index version in the background (memory-mapped or in heap), validate it (size sanity, sample queries), then **atomically swap a pointer** from old to new. In-flight requests finish on the old (ref-counted) and it is freed after. No downtime, no lock on the read path.
- Rollback = swap back to previous version.

**Trending layer**: small, fast, real-time overlay (recent-window counters) merged with the big static index at query time, so breaking news appears in minutes while the heavy index rebuilds hourly.

**Client and edge**
- Client **debounce** (~100-150 ms), cancel stale requests, cache results per prefix locally.
- CDN/edge cache for common short prefixes (a, ap, app...) with short TTL.
- Personalization: small per-user overlay (recent searches) merged client-side or in a thin service.

## 5. Stress test
| Failure | What happens | Mitigation |
|---|---|---|
| Bad index build (garbage/empty) | wrong suggestions | validation gates before publish, canary, instant rollback |
| Node OOM on load | crashes | memory-map, staged rollout, size checks |
| Kafka lag | stale trends | trending layer degrades gracefully; static index still serves |
| Offensive suggestion appears | trust/PR risk | blocklist at build and serve time, fast kill-switch |
| Traffic spike (event) | latency | in-memory serve is cheap; scale replicas; prefix CDN cache |
| Typo queries | poor results | fuzzy fallback/spell-correct path, separate from fast path |

## 6. Who pays
| Decision | Cost moves to |
|---|---|
| Precomputed top-K | Freshness (rebuild lag) |
| FST vs trie | Build complexity, harder updates (immutable) |
| Trending overlay | Extra system, merge logic |
| Client debounce | Slightly delayed first suggestion |
| Min-frequency filter | Rare but valid queries never suggested |

## 7. Drill questions
1. Why not `SELECT ... WHERE query LIKE 'pre%' ORDER BY count LIMIT 5`?
2. Why store top-K at each node rather than compute on lookup?
3. How do you swap a new 3 GB index in without dropping requests?
4. How does a breaking-news query get suggested within minutes?
5. How would you shard if the index no longer fit in one machine?

## 8. Practice notes (Java)
- Lucene's `FST`/`Suggester` classes are a real reference; Kafka Streams/Flink for windowed counts; `AtomicReference<Index>` is the swap in a few lines.

## 9. Further reading
- Lucene FST paper/blog posts (Mike McCandless); "Typeahead at LinkedIn/Twitter" engineering posts.
