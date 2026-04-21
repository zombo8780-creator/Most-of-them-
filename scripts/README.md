# Scripts

## `sync_from_swagger.py`

Diffs the Venice OpenAPI spec against the skills in this repo and flags drift.

```bash
python -m pip install pyyaml
python scripts/sync_from_swagger.py --spec https://api.venice.ai/doc/api/swagger.yaml
python scripts/sync_from_swagger.py --spec ./swagger.yaml --json > report.json
```

Exit code is `1` when drift is detected, so it can gate CI.

Reports:

- **`[NEW]`** — endpoint in the spec with no skill coverage.
- **`[STALE]`** — endpoint referenced in a skill but missing from the spec.
- **`[NEW VALUE]` / `[STALE VALUE]`** — enum drift for tracked enums (currently `/models?type=…`).

Expand `TRACKED_ENUMS` at the top of the script to track additional enums as new endpoints are added.
