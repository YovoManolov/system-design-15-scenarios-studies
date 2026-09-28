# 01. Distributed Rate Limiter & Abuse Defense

> Limit requests per client across a fleet of stateless servers, without the limiter becoming the outage.

## 1. Intent
- Protect backends from overload (fairness, cost control) **and** from abuse (credential stuffing, scraping). These are different problems with different signals.
- Clarify: limit per what (IP, API key, user, tenant)? Hard or soft limit? Is a 5% overshoot acceptable? What happens on limiter failure?

## 2. Numbers
- 1M req/s at the edge, 50M distinct keys, per-key limit e.g. 100 req/min.
- A Redis call adds ~0.3-1 ms in-region. At 1M rps, that is 1M Redis ops/s: needs sharding, or local pre-filtering.
- State per key (tokens + timestamp) is ~50-100 bytes: 50M keys is ~5 GB. Fits in memory.

## 3. Fault lines
1. **Check-then-decrement race**: two servers read "1 token left" and both allow. Needs an atomic operation.
2. **Hot keys**: one abusive client (or a big tenant) hammers a single Redis shard.
3. **Limiter in the hot path**: Redis latency or outage becomes request latency or outage.
4. **Clock skew**: refill computed from each app server's clock gives inconsistent behavior.

## 4. Design
**Algorithm choice**
- *Token bucket*: allows bursts up to capacity, refill at steady rate. Default choice.
- *Leaky bucket*: smooths output, no bursts.
- *Fixed window*: simple, but 2x burst at window boundary.
- *Sliding window log*: exact but memory heavy. *Sliding window counter* (weighted previous + current window) is the cheap approximation.

**Atomic token bucket in Redis (Lua)**: one script does read, refill, decrement, write, so it is atomic on the shard:
```
tokens, ts = HMGET key tokens ts
now = redis.call('TIME')            -- use Redis clock, not the client's
tokens = min(capacity, tokens + (now - ts) * rate)
if tokens >= cost then tokens -= cost; allowed = 1 else allowed = 0 end
HSET key tokens tokens ts now ; EXPIRE key ttl
return allowed
```
Key design: `rl:{tenant}:{route}` with hash tags so related keys land on the same slot when needed.

**Two tiers**
- **Local (Envoy local rate limit filter)**: in-process token bucket per instance. Zero network cost. Catches obvious floods.
- **Global (Envoy rate limit service, gRPC, backed by Redis)**: accurate cross-fleet limit. Called only for keys/routes that need it.
- Optional: servers **lease token batches** from Redis (take 50 tokens at once, spend locally). Trades accuracy for 50x fewer calls.

**Abuse defense layers** (limits alone don't stop attackers who rotate IPs)
- Per IP/ASN, per API key, per account, per device fingerprint.
- Graduated responses: allow, throttle (429 + `Retry-After`), challenge (CAPTCHA/proof-of-work), tarpit, shadow-ban.
- Login endpoints: limit per (account) and per (IP) separately, and count *failures*.

## 5. Stress test
| Failure | What happens | Mitigation |
|---|---|---|
| Redis shard down | limiter calls fail | **Fail-open** for normal APIs, **fail-closed** for login/payment. Circuit breaker + local fallback limit |
| Redis slow | request latency spikes | Tight timeout (5-10 ms), then fall back to local decision |
| Hot key | one shard saturated | Local pre-filter absorbs flood, split key into N sub-buckets (each gets limit/N) |
| Multi-region | each region enforces separately | Per-region budgets (limit split by traffic share), async reconcile; accept overshoot |
| Retry storm after 429 | clients retry immediately | `Retry-After`, exponential backoff with jitter, documented in SDK |

## 6. Who pays
| Decision | Cost moves to |
|---|---|
| Fail-open | Backends during a limiter outage |
| Local + leased tokens | Accuracy (overshoot), gain latency and cost |
| Strict global limit | Every request pays a network hop |
| Aggressive abuse rules | Legit users get false positives (support cost) |

## 7. Drill questions
1. Why is a Lua script needed and what would `GET` then `SET` break?
2. Redis is down. What does the limiter do for `/login` vs `/search`, and why?
3. How do you limit across 3 regions without a global Redis?
4. A tenant pays for 10k rps. How do you make sure one user in that tenant can't use all of it?
5. How do you tell abuse from a legitimate traffic spike?

## 8. Practice notes (Java / Spring)
- Bucket4j (with Redis/JCache backend) or Resilience4j `RateLimiter` for in-process; Spring Cloud Gateway has a Redis-based `RequestRateLimiter` (token bucket).
- Return `429` with `Retry-After` and `RateLimit-*` headers.

## 9. Further reading
- Envoy global rate limiting docs; Stripe engineering blog "Scaling your API with rate limiters"; Redis docs on Lua scripting/EVAL.
