# 12. Global Video Streaming & Transcoding

> Ingest an upload, transcode into many renditions in parallel, and serve smooth playback worldwide at minimal cost.

## 1. Intent
- Fast time-to-first-frame, minimal rebuffering, lowest possible bitrate per quality level, controlled cost.
- Clarify: VOD or live? Catalog size and long-tail distribution? DRM? Target devices/codecs? Upload-to-publish latency?

## 2. Numbers
- 500 hours of video uploaded per minute (YouTube scale). A 1 GB raw upload becomes ~8-10 renditions.
- Egress dominates cost: streaming 1B hours/day at ~3 Mbps average = ~1.35 EB/day. **CDN and codec efficiency are the cost levers.**
- Popularity is heavy-tailed: top few % of videos account for most views.

## 3. Fault lines
1. **Transcoding is CPU-heavy and slow** for long videos if done serially.
2. **Client bandwidth varies** during playback.
3. **Origin overload** if every viewer hits origin.
4. **Storage multiplies** by renditions and codecs.
5. **Encoding cost vs egress savings** for rarely watched vs viral content.

## 4. Design
**Upload -> processing pipeline (DAG)**
1. Resumable, chunked upload to object storage.
2. **GOP splitter**: split the source at **keyframe (GOP) boundaries** into independently decodable segments (e.g. 2-10 s). Because each segment starts on a keyframe, segments can be encoded independently and stitched without visual artifacts.
3. **Parallel transcoding**: each (segment x rendition) is a task on a worker pool via a DAG/workflow scheduler. A 2 hour movie finishes in minutes rather than hours. Tasks are idempotent; a dead worker's segment is simply retried.
4. **Packaging**: produce **HLS/DASH** with CMAF/fMP4 segments plus manifests listing renditions and segment URLs. Add thumbnails, captions, audio tracks, DRM (Widevine/FairPlay/PlayReady) as needed.
5. Publish manifest; content is served through the CDN.

**ABR (adaptive bitrate) ladder**
- Renditions e.g. 240p/400 kbps up to 4K/16 Mbps. **Per-title (and per-shot) encoding** picks the ladder from content complexity: cartoons need far fewer bits than sports at the same resolution.
- Client player picks the rendition per segment from measured throughput and buffer level (buffer-based algorithms like BBA avoid oscillation). Switching is possible because segments are aligned across renditions.

**Delivery**
- Tiered CDN: edge -> regional -> **origin shield** -> origin. Shield collapses misses so origin sees each segment once.
- **Anycast/DNS steering** routes users to a nearby healthy edge. Large streamers place caches inside ISPs (Netflix Open Connect) and **pre-position** popular titles off-peak (fill during the night).
- Long tail stays only at origin; popular content lives at the edge.

**Cost strategy**: encode with cheap codec (H.264) for everything, then spend expensive, slower encodes (VP9/AV1) only on videos that cross a popularity threshold, since higher compression saves egress at scale.

## 5. Stress test
| Failure | What happens | Mitigation |
|---|---|---|
| Transcode worker dies | segment missing | retry that task only; idempotent outputs |
| Edge cache cold (new viral video) | origin stampede | origin shield + request coalescing, prewarm |
| CDN PoP failure | viewers buffer | steering fails over to next PoP/CDN (multi-CDN) |
| Bandwidth drops mid-stream | rebuffer | ABR steps down; segments short enough to react |
| Corrupt segment | playback error | validate outputs (checksum, decode test) before publish |
| Codec unsupported by device | no playback | maintain fallback H.264 rendition |

## 6. Who pays
| Decision | Cost moves to |
|---|---|
| More renditions | Storage and encode compute |
| AV1 for popular titles | Encode CPU cost (and decode power on older devices) |
| Short segments | More requests, slightly lower compression efficiency |
| Parallel segment encoding | Slight quality seams; needs rate-control coordination |
| Long-tail off edge | Higher latency for rare content |

## 7. Drill questions
1. Why split on GOP boundaries and not fixed time slices?
2. How does the player decide when to switch quality?
3. What is an origin shield and what problem does it solve?
4. When is it worth spending on AV1 encodes?
5. How do you keep a viral upload from crushing the origin?

## 8. Practice notes
- Try it: `ffmpeg` to segment and produce an HLS ladder locally; inspect the `.m3u8` manifests.

## 9. Further reading
- Netflix tech blog (per-title encoding, Open Connect, Dynamic Optimizer); Apple HLS authoring spec; YouTube/Meta engineering posts on video processing.
