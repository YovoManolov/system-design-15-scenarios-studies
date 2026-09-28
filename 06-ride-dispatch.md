# 06. Real-Time Ride Dispatch & Proximity Matching

> Find the best nearby available driver for a rider in a couple of seconds, with locations that change every few seconds.

## 1. Intent
- Minimize rider wait (ETA), maximize driver utilization, never assign one driver to two riders.
- Clarify: greedy nearest-driver or batched global matching? Multiple offers at once? Pooling? Region size?

## 2. Numbers
- 1M active drivers, location update every 4 s: **250k writes/s**. Writes are tiny (id, lat, lng, ts, status).
- 50k ride requests/s peak city-wide is high; realistic per region ~1-5k/s.
- Location data is **ephemeral**: only the latest matters for dispatch. Durable history is for analytics, not dispatch.

## 3. Fault lines
1. **Proximity query** ("drivers within 2 km") over constantly moving points.
2. **Write volume** of location pings.
3. **Double-booking**: two dispatchers offer the same driver.
4. **Straight-line distance is not ETA** (rivers, one-way streets).
5. **Hot cells**: stadium letting out.

## 4. Design
**Spatial index: hex grid (H3)**
- Partition the map into hexagonal cells at a chosen resolution (e.g. ~res 8-9, neighbourhood scale). A hexagon has 6 equidistant neighbours, avoiding the diagonal-distance distortion of squares.
- Each driver maps to a cell id from their lat/lng. Index: `cell_id -> set of driver_ids`.
- **k-ring**: for a rider in cell C, search C plus rings 1..k around it, expanding until enough candidates. This solves the cell-boundary problem (nearest driver may be in the next cell).

**Ephemeral GPS store**
- Keep latest location **in memory** (Redis or custom in-memory service), keyed by driver, with TTL (a driver silent for ~30 s is considered offline).
- Do **not** write each ping to a durable DB on the hot path. Publish pings to Kafka for analytics/history.
- Shard by cell id (consistent hashing) so a proximity query touches few shards.

**Matching flow**
1. Rider request -> compute cell -> k-ring candidate drivers (available only).
2. Re-rank by **road-network ETA** (routing service), not Euclidean distance.
3. Offer to the best driver with a short timeout (~10-15 s). If declined/timed out, next.
4. **Reserve** the driver atomically (compare-and-set status `AVAILABLE -> OFFERED(ride_id, expires)`), so no other dispatch can offer them.
5. Accept -> `ON_TRIP`.

**Better matching**: batch requests in short windows (1-2 s) and solve an assignment problem globally (Hungarian/min-cost matching) instead of first-come greedy; reduces total wait at the cost of a small added delay.

**Hot cells**: dynamically use finer resolution or larger k only where density requires; surge pricing pulls supply/demand into balance.

## 5. Stress test
| Failure | What happens | Mitigation |
|---|---|---|
| Location shard down | drivers in region invisible | replicate, rebuild from next pings (state self-heals in seconds) |
| Two dispatchers offer same driver | double-booking | atomic CAS reservation on driver status |
| Driver goes offline mid-offer | rider waits | offer timeout, retry next candidate |
| GPS jitter / stale pings | bad ETA | TTL, smoothing, drop pings with old timestamp |
| Cell boundary | misses nearer driver | k-ring expansion |
| Regional failure | city down | region-isolated stacks; riders/drivers reconnect to region owner |

## 6. Who pays
| Decision | Cost moves to |
|---|---|
| Ephemeral in-memory locations | Lose history on crash (recomputed in seconds, and history goes via Kafka) |
| Finer cells | More cells to query per search |
| Batched matching | Small added latency for better global efficiency |
| ETA re-ranking | Extra routing calls (limit to top-N candidates) |

## 7. Drill questions
1. Why hexagons instead of geohash or a quadtree? What's the catch?
2. 250k location updates/s: where do they go and why not the main database?
3. How do you prevent one driver from receiving two offers?
4. Rider is on the edge of a cell. Walk through the search.
5. Greedy vs batched matching: when is each better?

## 8. Practice notes (Java / Spring)
- Uber's H3 has a Java binding (`com.uber:h3`). Redis `GEOSEARCH` is a simpler alternative to custom cell sharding for a first version.

## 9. Further reading
- Uber engineering: H3, "Ringpop", DISCO dispatch system; Lyft engineering on marketplace matching.
