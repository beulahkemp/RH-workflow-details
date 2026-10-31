# RUNBOOK — RH-workflow-details

## Purpose
Refresh the per-workflow detail corpus (D11) against the CURRENT catalog. Cost: ~2 requests per workflow (detail + creation list), ~140 min per shard at concurrency 6 on a GHA runner. Full matrix ≈ 2.5-3 h wall (4 shards parallel).

## When to refresh
- **Scheduled**: none standing (deliberate — this is a heavy walk; refresh on demand or after major catalog growth)
- **Trigger**: manual dispatch of `D11+J13 Enrichment Walk` on `comfy-backend/RH-workflow-runner` (workflow_dispatch)
- **Rule of thumb**: catalog grew >5% since last walk, or a quarterly cadence, whichever first

## Procedure
1. Open https://github.com/comfy-backend/RH-workflow-runner/actions/workflows/enrichment-walk.yml
2. "Run workflow" → inputs: `max_ids: 0` (full; a positive N = smoke subset), `concurrency: 6`
3. The walk is RESUMABLE — an existing corpus in `RH-workflow-details/data/` is the seed; only missing/new ids are fetched. Killed runs re-dispatch cleanly.
4. Monitor: each matrix leg logs a STATS line + heartbeat; the finalize job writes INDEX.json last.

## Exit criteria (verify after landing)
- [ ] `data/INDEX.json` `failed_shards` is empty
- [ ] total records == current catalog W-family rows (allow small drift for deletions mid-walk)
- [ ] `detail_gone + err` ratio < 1% (>1% = RH-side change; investigate before accepting)
- [ ] `md5`/`workflowContent` still null — if NON-null appears, RH re-opened the graph: re-enroll the workflowContent capture immediately (ledger J33 family)

## Rollback
The corpus is append/replace-by-id in git: `git revert` the refresh commit restores the prior corpus. Never force-push.

## Verify (quick local check)
```bash
git clone https://github.com/beulahkemp/RH-workflow-details.git && cd RH-workflow-details
python3 - <<'EOF'
import json, zstandard as zstd
dctx = zstd.ZstdDecompressor()
tot, ids = 0, set()
for i in range(4):
    with open(f"data/details-{i:02d}.jsonl.zst", "rb") as fh:
        with dctx.stream_reader(fh) as r:
            text = r.read().decode("utf-8")
    for line in text.split("\n"):          # \n only — NOT splitlines() (raw \u2028 in strings)
        if line.strip():
            rec = json.loads(line); tot += 1; ids.add(rec["id"])
print(tot, len(ids), "OK" if tot == len(ids) else "DUPES!")
EOF
```
Expect: `128011 128011 OK` (or the new row count after a refresh).
