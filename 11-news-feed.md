# 11. News Feed Fan-Out & Dynamic Ranking

> Show each user a fresh, relevant feed in <200 ms, when some users have 100M followers.

## 1. Intent
- Fresh + relevant + fast. Relevance is a ranking problem; freshness and latency are a fan-out problem.
- Clarify: chronological or ranked? Follow graph size distribution? Freshness SLA (seconds/minutes)? Ads/injections?

## 2. Numbers
- 500M DAU, each opens feed ~10x/day: ~58k feed reads/s average, several x at peak.
- Posts: 100M/day = ~1.2k/s. Average 200 followers: **~230k feed-list inserts/s** if we fan out on write.
- A celebrity with 50M followers: one post = 50M inserts. Unacceptable.

## 3. Fault lines
1. **Write amplification** for high-follower accounts.
2. **Read amplification** if we compute everything at read time (query N followees' posts per open).
3. **Inactive users**: pre-building feeds for people who never return wastes work.
4. **Ranking cost**: scoring thousands of candidates with a heavy model per request.

## 4. Design
**Push-pull hybrid**
- **Fan-out on write (push)** for normal accounts: on post, insert `post_id` into each follower's **precomputed feed list** (Redis sorted set/list, capped ~500-1000 entries). Read = fast fetch.
- **Fan-out on read (pull)** for celebrities (follower count > threshold): don't push. At read time, fetch recent posts from the celebrities the user follows and merge.
- Skip pushing to **inactive users**; rebuild their feed on next login.

**Merge at read time**: the user's precomputed list + N celebrity streams, each already time-sorted. Use a **k-way merge with a min-heap** (or max-heap by timestamp/score): push the head of each stream, pop best, push next from that stream. O(K log N) for top-K.

**Ranking pipeline (multi-stage funnel)**
1. **Candidate generation**: ~1-5k items from feed list + pull sources + recommendations.
2. **Light ranker** (cheap model, e.g. GBDT/small NN) trims to ~500.
3. **Heavy ranker**: deep model such as **DLRM** (Deep Learning Recommendation Model): sparse categorical features (user id, item id, author) go through **embedding tables**, dense features through an MLP, then explicit **feature interaction** (dot products), then top MLP predicts engagement probabilities (click, like, comment, hide).
4. **Re-rank / policy**: diversity, integrity/spam filters, freshness boosts, ads insertion.
- Final score = weighted combination of predicted actions (product decision).

**Feature/serving infra**: online **feature store** (low-latency reads) fed by streaming (Kafka) + offline batch; embedding tables are huge (sharded across parameter servers/GPUs); cache model outputs where safe; training/serving skew is a real risk.

**Storage**: posts in a sharded store keyed by `post_id`; feed lists store ids only (hydrate content at read; cache hot posts).

## 5. Stress test
| Failure | What happens | Mitigation |
|---|---|---|
| Celebrity posts | write storm | pull model for celebrities, threshold tuned |
| Redis feed cache lost | empty feeds | rebuild from follow graph + recent posts on read (pull fallback) |
| Ranker slow/down | latency/blank feed | timeouts, fall back to chronological or last cached ranking |
| Deleted/blocked post still in lists | policy violation | filter at hydration time, not just at write |
| Model quality regression | engagement drops | A/B, shadow scoring, gradual rollout, guardrail metrics |
| Follow/unfollow lag | stale feed | eventual; filter by current follow graph at read |

## 6. Who pays
| Decision | Cost moves to |
|---|---|
| Push for most users | Storage and write throughput |
| Pull for celebrities | Read latency and merge complexity |
| Heavy model | GPU/CPU cost, latency; funnel limits it |
| Ranking by engagement | Content quality/integrity concerns (product tradeoff) |
| Skipping inactive users | Slower first load after return |

## 7. Drill questions
1. Push vs pull: which for a user with 50M followers and why? What's the threshold based on?
2. How do you merge 20 time-sorted streams efficiently?
3. Why a multi-stage ranking funnel instead of one big model?
4. What is in a DLRM and why are embeddings the scaling problem?
5. A post is deleted. How does it disappear from feeds?

## 8. Practice notes
- Build a toy: Redis ZSET feed + Java `PriorityQueue` merge over streams. Easy to explain in an interview.

## 9. Further reading
- Twitter "Timelines at scale" (Raffi Krikorian); Meta DLRM paper (2019) and open-source repo; Instagram engineering on feed ranking.
