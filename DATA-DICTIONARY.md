# DATA-DICTIONARY — RH-workflow-details

Auto-derived from the actual corpus (2026-10-23 walk, verified 2026-10-31 R11: 128,011 records, 128,011 distinct ids, 0 dupes). Every field the API returned is preserved verbatim. Coverage measured across all 128,011 records.

## Envelope (`data/details-XX.jsonl.zst`, one JSON object per line)

| field | type | coverage | notes |
|---|---|---|---|
| `id` | str | 100% | workflow id (snowflake); join key to `RH-workflow-data` W-shards |
| `domain` | str | 100% | `"ai"` (98,294) or `"cn"` (29,717) — which site served the detail |
| `detail` | object | 99.88% | the `/api/portal/workflow/detail` `data` payload, verbatim; absent on dead records |
| `err` | int | 0.12% (154) | present ONLY on dead records instead of `detail`: the API code (`404`=deleted, `412`, `1913`, `2020`, `500`) |

## `detail.*` (the API payload — 127,857 records with it)

| field | type | non-empty | notes |
|---|---|---|---|
| `id` | str | 100% | mirrors envelope id |
| `name` | str | 99.9% | display name (zh or en) |
| `desc` | str | ~99% | HTML (`<p>…`) description, may embed images |
| `systemWorkflow` | bool | 100% | RH-official flag |
| `publishTime` | str | 99.9% | ISO-8601 with offset; span 2024-04-18 .. 2026-10-23 |
| `timestamp` | str | ~100% | epoch-millis as string |
| `md5` | null | **0%** | always null (see README finding) |
| `imageSize` | str\|null | rare | preset size label |
| `preview` | object\|null | ~50% | `{id, objName, url, imageWidth, imageHeight}` — poster/preview image on the public CDN |
| `owner` | object | ~100% | `{id, avatar, name}` — public display data (NOTICE class) |
| `workflowContent` | null | **0%** | always null — the graph is not anonymously fetchable |
| `nodeCount` | int | 99.9% | total nodes |
| `statisticsInfo` | object | ~100% | `{likeCount, downloadCount, useCount, pv, collectCount}` — all STRING-typed numbers |
| `labels` | str | ~90% | comma-joined zh label names |
| `tags` | list | ~30% | string tags |
| `covers` | list | ~60% | `[{id, url, thumbnailUri, objName, imageWidth, imageHeight}]` |
| `chineseCovers` / `englishCovers` | list | sparse | localized cover variants |
| `primitiveNodes` | list[str] | 95.0% | built-in ComfyUI node CLASS_NAMES (e.g. `CLIPTextEncode`) |
| `customNodes` | list[str] | 99.4% | custom node class names (e.g. `ImageUpscaleWithModel`) — the D11 join key for custom-node census |
| `usedModels` | list[str] | 83.5% | model FILE names (e.g. `turbovisionxlSuperFastXLBasedOnNew_tvxlV431Bakedvae.safetensors`) — the D11 join key for model census |
| `liked` / `likeCount` / `collect` / `collectCount` | bool/str | 100% | anonymous-viewer state + display counts (strings) |
| `webappId` | str | 64.3% | join key to the AI-app gallery (`RH-gen-surface` apps corpus) |
| `status` / `workflowState` / `webappState` | int | varies | publish state enums |
| `publishAccess` | object | ~100% | `{accessType, publishScope, encryptFlag?}` access enums |
| `canModifyPublishScope` / `canModifyPublishType` | bool | ~100% | owner-plane flags (false anonymously) |

## INDEX.json / stats-XX.json

| field | notes |
|---|---|
| `detail_ok` / `detail_gone` / `detail_err` | walk outcomes (DELTA for the run when resumed — the jsonl is the full state) |
| `wf_with_creations` / `creation_records` / `creation_overflow_wf` | creation-list outcomes per shard |
| `catalog_rows` / `resumed_from` / `walked_this_run` | resume bookkeeping |
| `requests` / `transport_errors` / `elapsed_min` | cost telemetry |
| `run_id` | the GHA run that wrote it |

## Verified invariants (re-check on any refresh)
- records == distinct ids == catalog W-family rows (0 missing, 0 dupes)
- `err` records never carry a `detail`; `detail` records never carry `err`
- `md5` and `workflowContent` null across the corpus (if this CHANGES, RH re-opened the graph — re-enroll immediately)
