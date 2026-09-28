# 09. Cloud File Storage & Chunked Sync (Dropbox-style)

> Sync large files across devices efficiently, deduplicate storage, resolve conflicts, never lose a byte.

## 1. Intent
- Correctness: no data loss, files converge across devices. Efficiency: upload/download only what changed.
- Clarify: max file size? Sharing/permissions? Versioning/history? Client-side encryption? Team folders?

## 2. Numbers
- 500M users, avg 5 GB = 2.5 EB logical. Dedup across chunks cuts physical storage materially.
- Edits are usually small changes to big files: a 1 GB file with a 1 MB edit should transfer ~MBs, not GB.
- Metadata ops (list, stat, sync cursor) dominate request count; blob traffic dominates bytes.

## 3. Fault lines
1. **Fixed-size chunking breaks on insertion**: insert 1 byte at the start and every chunk boundary shifts, so everything re-uploads.
2. **Atomic file version commit** across many chunks.
3. **Duplicate chunks** across files/users wasting storage.
4. **Concurrent edits** from two devices.
5. **Garbage collection** of chunks no file references.
6. **Cross-user dedup leaks information** (you can test if someone has a file).

## 4. Design
**Split control plane and data plane**
- **Metadata service** (sharded relational DB, by user/namespace): file tree, versions, chunk manifests. Strong consistency needed.
- **Block store**: immutable chunks in object storage (S3), addressed by content hash.

**Content-defined chunking (CDC)**
- Boundaries are chosen from the *content* via a rolling hash (Rabin; **FastCDC** uses a faster gear hash), e.g. average chunk 1-4 MB with min/max bounds. An insertion only changes the chunks around it; the rest keep the same boundaries, so hashes match and are skipped.
- Chunk id = `SHA-256(chunk bytes)`.

**Upload flow**
1. Client chunks file locally, computes hashes -> manifest `[h1, h2, h3, ...]`.
2. Client asks server: "which of these hashes do you not have?" (`need_blocks`).
3. Client uploads only missing chunks directly to object storage (pre-signed URLs; resumable, parallel).
4. Client **commits**: metadata transaction creates a new file version = ordered list of chunk hashes. Commit is atomic; readers see either the old version or the new, never a mix.
- Chunks are **immutable and content-addressed**, so re-uploading the same chunk is idempotent, and retries are safe.

**Sync**
- Server keeps a per-namespace **change journal** with a monotonically increasing cursor. Clients long-poll/subscribe "changes since cursor N" and apply. Local file watchers detect edits, debounce, then chunk.

**Conflicts**
- Version check on commit (`base_version`). If the base is stale, keep both: create a "conflicted copy" file. Never silently overwrite.

**Deletion and GC**
- Reference counting or periodic mark-and-sweep over manifests; delay deletion (grace period) because a commit may be in flight referencing a "unreferenced" chunk.

**Dedup security**
- Global dedup lets an attacker confirm a file exists by uploading its hash. Mitigate: require proof-of-possession, scope dedup per account/team, or accept cross-user dedup only for large, common blocks with checks.

## 5. Stress test
| Failure | What happens | Mitigation |
|---|---|---|
| Upload interrupted | partial chunks | chunks are independent and idempotent; resume; unreferenced chunks are GC'd later |
| Commit races with GC | version points to deleted chunk | GC grace period, mark-and-sweep with in-flight protection |
| Two devices edit offline | divergent versions | conflicted copy on commit |
| Hash collision | wrong data | SHA-256 makes it negligible; verify size + hash |
| Metadata shard down | namespace unavailable | replicas, failover; data plane is unaffected for cached clients |
| Client uploads garbage claiming a hash | poisoned dedup | server verifies hash of uploaded bytes before accepting |

## 6. Who pays
| Decision | Cost moves to |
|---|---|
| CDC | Client CPU (rolling hash) and complexity |
| Global dedup | Privacy risk; saves storage cost |
| Small chunks | More metadata, more requests; better dedup |
| Conflict copies | Users must merge manually |
| Immutable blocks | GC complexity |

## 7. Drill questions
1. Why not fixed 4 MB chunks? Demonstrate with a 1-byte insertion.
2. Walk through uploading a 2 GB file where 1 MB changed.
3. How is a multi-chunk commit atomic?
4. When is it safe to delete a chunk?
5. What's the privacy problem with cross-user dedup?

## 8. Practice notes
- Implement a tiny CDC chunker in Java/Go with a gear hash and test boundary stability on inserts. Good weekend project and interview story.

## 9. Further reading
- FastCDC paper (USENIX ATC 2016); Dropbox "Streaming File Synchronization", "Rewriting the heart of our sync engine"; restic/borg docs on chunking.
