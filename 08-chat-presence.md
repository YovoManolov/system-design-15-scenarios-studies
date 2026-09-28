# 08. Collaborative Chat & Presence at Scale

> 10M concurrent WebSocket connections, ordered messages, and "who's online" without melting the fleet.

## 1. Intent
- Deliver messages in order, exactly once *to the user's view*, with low latency; show presence cheaply.
- Clarify: 1:1, small groups, or huge channels? Message history and sync across devices? Read receipts? Is presence precise or approximate?

## 2. Numbers
- 10M concurrent connections. A tuned box handles ~250k-1M idle connections (memory ~10-50 KB/conn incl. buffers): **~40 gateway nodes at 250k each**.
- Each user sends ~20 messages/day: 10M active x 20 = 200M/day = ~2.3k msg/s average, 10x peaks.
- Presence heartbeats every 30 s from 10M clients = **333k heartbeats/s**. This is bigger than the message load, which is why presence needs special design.

## 3. Fault lines
1. **Connection state is sticky**: a message for user U must reach the node holding U's socket.
2. **OS limits**: file descriptors, memory per connection, kernel buffers.
3. **Presence fan-out explosion**: user with 1,000 contacts going online notifies 1,000 people.
4. **Reconnect storm** after a gateway or region failure.
5. **Ordering and delivery guarantees** across devices and retries.

## 4. Design
**Connection gateways**
- Event-driven servers (epoll/Netty, not thread-per-connection). Tune: `ulimit -n`, `net.core.somaxconn`, TCP buffer sizes, `tcp_keepalive`. Keep per-connection state tiny.
- Gateways are dumb: auth, heartbeat, forward. No business logic, so they rarely need deploys (deploys drop connections).

**Routing**
- **Session registry** (Redis): `user_id -> {gateway_id, conn_id}` per device, with TTL refreshed by heartbeat.
- Message path: sender -> gateway -> **chat service** -> persist -> lookup recipients' gateways -> publish to that gateway's channel (Kafka topic or Redis Pub/Sub per gateway) -> gateway pushes to socket.
- Recipient offline: store, then send a push notification.

**Ordering and reliability**
- Each conversation has a monotonically increasing **sequence number** assigned by the conversation's owner shard (partition by `conversation_id`).
- Client sends a client-generated `message_id` (idempotency, dedupes retries), server acks with the sequence.
- Delivery is at-least-once; client dedupes by id and detects gaps by sequence; on reconnect it asks "give me everything after seq N".
- Storage: wide-column or sharded SQL keyed by `(conversation_id, seq)`. Reads are range scans by conversation.

**Large groups**: small groups fan-out on write to member gateways; huge channels use **pull/subscribe** (clients fetch from the channel log) so a message isn't multiplied by 100k.

**Lazy presence**
- Don't broadcast every state change. Store `last_seen` / online with TTL.
- **Compute on demand**: clients subscribe to presence only for **visible** contacts (open chat, roster on screen). Publish presence changes only to current subscribers.
- **Debounce flapping** (mobile network blips): treat as offline only after N seconds without heartbeat.
- Coarse resolution ("online / seen 5 min ago") instead of real-time.

**Reconnect storms**: jittered exponential backoff on clients; gateway connection-rate limits; drain gateways gracefully on deploy (tell clients to reconnect to elsewhere over time).

## 5. Stress test
| Failure | What happens | Mitigation |
|---|---|---|
| Gateway crash | ~250k users disconnect | clients reconnect with jitter; sync from last seq; registry TTL cleans stale entries |
| Registry stale (points to dead gateway) | message not delivered live | delivery ack/timeouts, fall back to store + push, client sync on reconnect |
| Duplicate send (client retry) | double message | client message_id dedupe |
| Out-of-order arrival | UI shows wrong order | order by conversation seq, not arrival time |
| Presence flood | CPU spent on heartbeats | subscribe-only presence, coarse TTLs, batch heartbeats |
| Region outage | mass reconnect | multi-region, capacity headroom, connection admission control |

## 6. Who pays
| Decision | Cost moves to |
|---|---|
| Lazy/coarse presence | Accuracy: "online" may be stale by seconds |
| Per-conversation sequencing | One shard is the ordering point (hot for giant channels) |
| Dumb gateways | An extra hop to the chat service |
| Sticky connections | Deploys are disruptive; need drain logic |

## 7. Drill questions
1. How does a message for user U find U's socket among 40 gateways?
2. What actually limits connections per machine, and how do you raise it?
3. Why is presence harder than messaging here, and how do you fix it?
4. A gateway dies. What do 250k clients do, and how do you stop a stampede?
5. How do you guarantee ordering and no duplicates?

## 8. Practice notes (Java / Spring)
- Spring WebFlux/Reactor Netty or plain Netty for gateways (Spring MVC + Tomcat threads won't hold 250k sockets). STOMP over WebSocket is fine for small scale.

## 9. Further reading
- WhatsApp/Erlang scaling posts ("1 million connections"), Discord engineering on presence, Slack "flannel"/edge caching.
