# 8. Results store

Object storage with a run index now; a results database later (D14). Written only by the engine and the executor; read by the optimizer and by people.

## 8.1 Layout

```
<service>/<yyyy-mm-dd>/<run-id>/
  run.json
  results.json
  logs/<case-id>[<dataset>].jsonl
  report.md
  analysis.json
  evidence/
index/runs/<yyyy-mm>/<run-id>.json
sessions/<yyyy-mm-dd>/<session-id>.json
optimizer/<service>/<yyyy-mm-dd>/R-<yyyymmdd>-<n>.md
```

## 8.2 Run index

One small object per run under `index/runs/<yyyy-mm>/`, so parallel writers never touch the same object (EX-8, ST-4). The local folder connector and the object storage connector answer `query()` by listing the month prefixes in range and filtering. When dashboards need it, a results database loads these objects; the connector interface does not change.

```json
{
  "run_id": "run-20261115T1830Z-b41c",
  "service": "payments", "env": "qa", "apis": ["authorize", "refund"],
  "started": "2026-11-15T18:30:02Z", "finished": "2026-11-15T18:31:47Z",
  "plan_version": "1.2", "framework": "1.4.0", "test_assets_commit": "e3d90f1",
  "deployed_service": "2.3.7",
  "counts": {"passed": 1, "failed": 1, "pending_deploy": 3, "blocked": 1, "skipped": 0},
  "trigger": "manual", "session_id": "ses-20261115T1830Z-9f2c",
  "location": "payments/2026-11-15/run-20261115T1830Z-b41c/"
}
```

## 8.3 run.json

| Field | Meaning |
|---|---|
| `run_id`, `service`, `environment`, `apis` | What ran |
| `trigger`, `session_id` | Who or what started it |
| `versions` | `framework`, `test_assets_commit`, `plan`, `sections` (per API), `deployed_service`, `version_unknown`, `assumed_version`, `validators` (name to version) (CV-6) |
| `started`, `finished` | UTC timestamps |
| `selection` | Filters used and case count |
| `summary` | Counts per status, plus `quarantined_failed` |
| `overrides` | Whether a `.local` data file was used (names only) |
| `skipped_steps` | From the session (SF-5) |
| `agent` | Session reference and cost of the executor's agent steps |

## 8.4 results.json

One entry per case and data set:

```json
{
  "case": "PAY-AUTH-TC-0002", "dataset": null, "status": "failed",
  "failure_kind": "validation", "step": "authorize_over_limit",
  "validators": [
    {"name": "http_status", "validation": "authorize.V2", "passed": true},
    {"name": "error_code", "validation": "authorize.V2", "passed": false,
     "expected": "DECLINED", "observed": "LIMIT_EXCEEDED", "message": "body.code: expected DECLINED, got LIMIT_EXCEEDED"}
  ],
  "duration_ms": 1840, "trace_ids": ["a3ce929d0e0e4736"],
  "stale": false, "quarantined": false,
  "log": "logs/PAY-AUTH-TC-0002.jsonl"
}
```

`pending_deploy`, `blocked`, and `skipped` entries carry `reason` instead of validator results.

## 8.5 Logs

JSON Lines, one event per line, masked before writing: `request`, `response`, `retry`, `save` (names only), `validate`, `teardown`, `note`. Each line has `ts`, `case`, `dataset`, `step`, `event`, and event fields. Bodies over 256 KB are written to `evidence/` and referenced by path.

## 8.6 Publishing

The executor's `publish` step:

1. Pushes `results.json` to the `test_management` connector, mapping case ids to the tool's items (results only, D11).
2. Posts the headline and counts with `results_store.link(location)` to each target in `agent.yaml` `publish.targets` (a PR comment through `ci_cd`, or a chat channel through a plugin) (ST-6, EX-14).

A publish failure is recorded in the session and does not change the run.

## 8.7 Retention

Per environment from `defaults.retention_days` in the firm profile, overridable per service. The index entry outlives the run folder by one year, so trends stay queryable after logs expire; `location` then points to nothing and the optimizer treats the run as counts-only.
