# RH-workflow-details — the per-workflow detail corpus (D11)

The full-detail enrichment of the workflow catalog: **128,011 records = 127,857 live detail payloads + 154 dead/unfetchable markers**, walked 2026-10-23 (GHA run 39892997047, 4-way matrix, ~240k requests, zero dupes, resumable). This is the corpus that joins the catalog rows (`RH-workflow-data` W-family shards) to their node/model/app linkage — the thing the daily pipeline never persisted (raw details lived only in ephemeral Actions cache).

## What this repo is
- **Kind**: workflow detail records (ONE kind — the by-kind discipline; joins happen at your consumption layer)
- **Created**: 2026-10-23 (R10 dispatch / R11 landing; see `RH-workflow/audit/RESOURCE-LEDGER.md` D11)
- **Upstream**: RunningHub `/api/portal/workflow/detail` (GET, anonymous, per-domain Origin/Referer — see PROVENANCE for the .cn Origin-matching trap)
- **Auth**: anonymous public reads
- **Complete vs catalog**: exactly the 128,011 W-family rows of the 2026-10-23 catalog refresh (0 missing, 0 duplicated ids)

## Repo layout
| Path | Content |
|---|---|
| `data/details-XX.jsonl.zst` | 4 shards, JSONL (zstd-19), one record per workflow id |
| `data/INDEX.json` | merged walk stats (per-shard + totals, run id) |
| `data/stats-XX.json` | per-shard walk stats (DELTA semantics on resume — the jsonl is the truth) |
| `data/splits-XX.json` | poison-burst split records (empty `[]` = no splits taken) |

## Headline numbers (2026-10-23 walk)
| Metric | Value |
|---|---|
| Records / distinct ids | 128,011 / 128,011 (0 dupes) |
| With detail payload | 127,857 (99.88%) |
| Dead markers (`err`) | 154: 404×109 (deleted), 412×20, 1913×18, 2020×3, 500×4 |
| Domain split | .ai 98,294 / .cn 29,717 |
| publishTime span | 2024-04-18 .. 2026-10-23 |
| `customNodes` non-empty | 127,209 (99.4%) |
| `primitiveNodes` non-empty | 121,582 (95.0%) |
| `usedModels` non-empty | 106,878 (83.5%) |
| `webappId` set | 82,261 (64.3% — join key to the AI-app gallery) |
| `workflowContent` / `md5` | **null corpus-wide** — see the finding below |

## THE workflowContent finding (2026-10-23)
The detail API returns `"workflowContent": null, "md5": null` for **every** record (0/128,011 non-null). The workflow JSON graph is **no longer anonymously fetchable** from this endpoint (RH removed it from the anonymous payload; historical canvas-export evidence had it via editor-class surfaces). What survives anonymously: the node inventories (`primitiveNodes`, `customNodes`), `usedModels`, `nodeCount`, covers, stats. Graph capture would need the editor/JWT class (ledger J33 family). Do not "re-walk to fill workflowContent" — it is not there to fill.

## Reading the shards (two traps)
1. **Split on `\n` only, never `splitlines()`** — some strings contain raw `\u2028`/`\u2029` (LINE SEPARATOR) inside values; Python's `splitlines()` breaks those lines mid-string and the JSON fails to parse.
2. Decompress with a streaming zstd reader; shard 00 alone is 138MB raw / 8.3MB zst.

```python
import json, zstandard as zstd
dctx = zstd.ZstdDecompressor()
with open("data/details-00.jsonl.zst", "rb") as fh:
    with dctx.stream_reader(fh) as r:
        text = r.read().decode("utf-8")
records = [json.loads(l) for l in text.split("\n") if l.strip()]
```

## Downstream consumption
Consume **by kind**; field meanings: `DATA-DICTIONARY.md` (auto-derived from the actual data — every field the API returned is preserved verbatim). Refresh procedure: `RUNBOOK.md`.
