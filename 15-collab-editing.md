# 15. Collaborative Document Editing & Real-Time Sync

> Multiple people type in the same document at once, everyone sees the same result, and nobody's intent is lost.

## 1. Intent
- **Convergence** (all replicas end in the same state) and **intention preservation** (an insert lands where the user meant it, even if others edited concurrently). Low latency for local typing.
- Clarify: rich text or plain? Offline editing? Number of concurrent editors per doc? Version history/undo? Central server allowed or peer-to-peer?

## 2. Numbers
- Typical doc: 2-10 editors; worst case a few hundred (all-hands doc). Ops are tiny (insert/delete of a character or run).
- Per doc: ~5-20 ops/s per active typist => hundreds of ops/s at most. **The hard problem is correctness, not throughput.**
- Millions of docs: each doc is an independent unit, so shard trivially by `doc_id`.

## 3. Fault lines
1. **Concurrent edits at overlapping positions** (two users insert at index 5).
2. **Latency**: local edit must apply immediately (optimistic), reconciled later.
3. **Ordering/causality** across replicas with network reordering.
4. **Metadata growth** (tombstones/IDs) over long-lived documents.
5. **Offline divergence** and merging.

## 4. Design
**Option A: Operational Transformation (OT)** (classic Google Docs approach)
- Operations are position-based: `insert(5, "x")`. When a concurrent op arrives, **transform** it against ops it didn't see so its positions still mean the same thing.
- Example: Alice `insert(3,"A")`, Bob `delete(5)` concurrently. Bob's delete, applied after Alice's insert, must become `delete(6)`.
- **Central server** gives a total order: each op is stamped with a revision. Client keeps *inflight* (sent, unacked) and *buffered* ops; server transforms incoming against history since the client's base revision. Simple correctness argument thanks to the server, but transformation functions for rich text are notoriously hard to get right.

**Option B: CRDTs** (Yjs, Automerge; sequence CRDTs like RGA, Logoot, YATA)
- Every character/element gets a **globally unique ID** (e.g. `(client_id, counter)`) and a reference to its neighbours (e.g. "after element X"). Insert = add element with that anchor; delete = mark as **tombstone**.
- Operations are **commutative and idempotent**: any order, duplicates OK, replicas converge without a central arbiter. Great for offline and peer-to-peer.
- Costs: per-element metadata (memory), tombstones accumulate (need GC/compaction once all peers have seen a delete), more complex undo.

**Causality: version vectors / vector clocks**
- To know what a replica has seen, track `{client_id: highest_counter}`. Compare vectors to decide "A happened before B" vs "concurrent". Used to request missing ops on sync and to ensure causal delivery (an op that depends on an unseen element is buffered).
- Lamport clocks give a total order but not concurrency detection; vector clocks do.

**System architecture**
- **Session server per document**: route all editors of `doc_id` to the same node (consistent hashing/sticky), which holds the doc in memory, orders/relays ops over WebSocket, and persists.
- **Persistence**: append-only op log + periodic **snapshots** (load = latest snapshot + tail of ops). Compact old ops.
- **Presence/cursors** are ephemeral: broadcast, don't persist, throttle.
- **Undo** is per-user (invert *your* ops, transformed against everyone else's).

## 5. Stress test
| Failure | What happens | Mitigation |
|---|---|---|
| Session server crash | active editors disconnect | clients reconnect, resend unacked ops with ids (idempotent), reload from snapshot + log |
| Duplicate op delivery | double insert | op ids (client_id, seq) dedupe; CRDT ops idempotent |
| Out-of-order ops | wrong positions | causal buffering via vector clocks/revisions |
| Long offline session | huge merge | CRDT merge handles it; OT needs history since base (may reject/rebase) |
| Tombstone growth | memory bloat | GC after all peers acked, or periodic "checkpoint" doc rewrite |
| Malicious/buggy client | corrupt doc | server validates ops, permission checks, size limits |

## 6. Who pays
| Decision | Cost moves to |
|---|---|
| OT + central server | Engineering complexity of transform functions; server is a required arbiter |
| CRDT | Memory/metadata overhead and tombstone GC |
| Sticky doc sessions | Doc-level hotspot; failover complexity |
| Snapshots | Storage and compaction jobs |
| Optimistic local apply | Occasional visible reordering when reconciling |

## 7. Drill questions
1. Alice inserts at 3, Bob deletes at 5, concurrently. Show how OT fixes Bob's op.
2. Why do CRDTs need unique IDs and tombstones?
3. Vector clock vs Lamport clock: what can one do that the other can't?
4. Why route all editors of a document to one server?
5. When would you choose OT, and when a CRDT?

## 8. Practice notes
- Play with Yjs (JS) or Automerge; read Yjs internals (YATA). In Java, look at the OT-based `ShareJS`-style ports or model a tiny RGA CRDT yourself.

## 9. Further reading
- Jupiter/Google Wave OT papers; Martin Kleppmann "CRDTs: The Hard Parts" talk and Automerge paper; Yjs docs.
