# PROVENANCE — RH-workflow-details

## Source chain
- **Catalog seed**: `beulahkemp/RH-workflow-data` W-family shards (the 2026-10-23 refresh; 128,011 workflow rows with `source_domain`)
- **Detail endpoint**: `GET https://www.runninghub.ai/api/portal/workflow/detail?id=<wf_id>` (and the `.cn` twin) — anonymous, JSON
- **Walker**: `RH-workflow/scripts/r11/r11_enrichment_walk.py` (in the main repo; adversarially reviewed R10 — F1..F17 folded: seed-merge resume, poison-burst breaker, parts-aware resume, per-shard finalize)
- **Compute**: GitHub Actions matrix (`comfy-backend/RH-workflow-runner/.github/workflows/enrichment-walk.yml`), 4 shards × ~140 min, concurrency 6 — the 15-30-min sandbox rule means this walk ALWAYS runs on GHA, never locally
- **Walk of record**: run 39892997047 (2026-10-23), preceded by smokes 39886475230 (family-shard discovery), 39889218331 (resume-step exit trap), 39890553401 (green)

## Request shape (reproducible)
```
GET /api/portal/workflow/detail?id=<id>
Headers: User-Agent: Mozilla/5.0; Origin/Referer MUST match the target domain
```

**The .cn Origin trap (discovered R10, J06)**: `.cn` endpoints enforce Origin-matching and reject `.ai`-origin requests with a MISLEADING body `模板不存在` ("template not exist") that looks like a 404. Per-domain Origin/Referer is mandatory on any dual-domain walk.

## Dead-record semantics
154 ids returned no detail. Codes: `404` (109 — deleted between catalog snapshot and walk), `412` (20), `1913` (18), `2020` (3), `500` (4). These are recorded as `{"id", "domain", "err"}` markers so a future refresh can distinguish "still dead" from "newly revived".

## Resumption contract
The walk is idempotent + resumable: existing shard records are the seed; only missing ids are fetched; results merge by id (disjoint by construction). A killed matrix re-dispatch loses nothing (smoke #3's 8k records were carried into the full run by exactly this mechanism).

## Capture integrity
- zstd-19 JSONL; INDEX.json + stats-XX.json written by the finalize job after all shards land
- no transformation of API payloads: `detail` is the response `data` verbatim
- no PII beyond public display fields (owner name/avatar — NOTICE class)

## Known limitations
- `workflowContent`/`md5` null corpus-wide (API-side; see README) — node/model inventories survive, the graph does not
- creations for these workflows live in `RH-community/data/creations/` (by-kind discipline), keyed by the same workflow ids
