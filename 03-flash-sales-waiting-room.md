# 03. Flash Sales & Virtual Waiting Rooms (Ticketmaster)

> 2M people want 50k seats at the same second. Nobody gets oversold, the system survives, and it feels fair.

## 1. Intent
- Goals: **never oversell**, **don't fall over**, **be perceived as fair**, resist bots.
- Clarify: general admission or reserved seats? Is FIFO required or is a lottery fine? How long can someone hold a seat while paying?

## 2. Numbers
- 2M users arrive within 60 s: ~33k new sessions/s, each polling status. Checkout capacity is maybe 2k/s.
- 50k seats, ~10 min hold. If 1M people hit "buy" against a database directly, you get row lock storms.

## 3. Fault lines
1. **Thundering herd** on the seat inventory.
2. **Hot-row contention**: everyone updates the same rows or the same counter.
3. **Oversell** from check-then-act races.
4. **Bots** win every time unless you stop them.
5. **Abandoned holds** lock inventory that real buyers could take.

## 4. Design
**Layer 1: Virtual waiting room (protects the system)**
- Sits at the CDN/edge, served as static + a tiny status API. Users get a **queue position or random lottery ticket** on arrival.
- Fairness: for a fixed on-sale time, randomize everyone who arrives in the first N minutes (a pure FIFO rewards fastest network and bots).
- **Admission rate = downstream capacity.** A controller admits X users per second, matched to checkout throughput and error rate (adaptive).
- Admitted users receive a signed **admission token** (JWT: user/session, expiry, event id). Checkout endpoints reject requests without a valid token. This is enforced at the edge/gateway, cheaply.
- Client polls with jittered interval, or use SSE/WebSocket for position updates. Don't let 2M clients hammer `/status` every second.

**Layer 2: Inventory correctness**
- Seat states: `AVAILABLE -> HELD(user, expires_at) -> SOLD`.
- Hold with an atomic conditional write:
  - SQL: `UPDATE seat SET status='HELD', holder=?, expires_at=? WHERE id=? AND status='AVAILABLE'` (check affected rows = 1).
  - Or Redis: `SET seat:{id} user NX PX 600000`.
- General admission (counter, not seats): shard the counter into N buckets (`stock:{sku}:{0..N}`), decrement a random bucket, fall through to others. Removes the single hot row.
- Payment success converts HELD to SOLD in a transaction; timeout/expiry returns it to AVAILABLE (a sweeper or Redis TTL plus reconciliation).

**Layer 3: Anti-bot**
- Per-account/payment-method purchase limits, device/behavior signals, CAPTCHA at queue entry, verified-fan pre-registration.

## 5. Stress test
| Failure | What happens | Mitigation |
|---|---|---|
| Two users grab same seat | oversell | conditional update / `SET NX`; DB is source of truth |
| Payment slow, hold expires mid-payment | user pays for a seat someone else now holds | extend hold when payment starts; on payment success, re-check hold ownership, refund on loss |
| Redis lost | holds lost | rebuild from DB; DB has authoritative `HELD` rows with expiry |
| Admission token shared/resold | bots bypass queue | bind token to session + device, short TTL, single use |
| Queue service crash | users lose place | persist positions (Redis/DB), tokens are stateless so admitted users are unaffected |

## 6. Who pays
| Decision | Cost moves to |
|---|---|
| Waiting room | Users wait (and complain), but backend survives |
| Lottery instead of FIFO | Fastest users lose their advantage; better perceived fairness |
| Short hold time | Slow payers lose seats |
| Aggressive anti-bot | False positives on real fans |

## 7. Drill questions
1. Why not just scale the checkout service horizontally?
2. Two users click the same seat at the same millisecond. Trace exactly what happens.
3. How does the admission controller decide how many people to let in?
4. How do you keep 2M clients from polling your status endpoint to death?
5. General admission with a single `stock` row is a bottleneck. Fix it, and explain what breaks when stock is nearly zero.

## 8. Practice notes (Java / Spring)
- Optimistic locking (`@Version`) works for low contention only; under a flash sale prefer conditional `UPDATE` or Redis atomic ops.
- Admission token: sign with HMAC/RS256, verify at Spring Cloud Gateway filter.

## 9. Further reading
- Ticketmaster/Queue-it engineering write-ups on virtual waiting rooms; Cloudflare Waiting Room docs.
