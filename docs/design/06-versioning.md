# 6. Versioning and change tracking

This chapter settles every TBD in [versioning.md](../versioning.md) and section 8 of the requirements.

## 6.1 Fingerprints

A fingerprint is the first 12 hex characters of the SHA-256 of a canonical form of the content. Canonical forms remove changes that do not change meaning, so edits to wording in a description do not mark cases stale.

| Source | Canonical form |
|---|---|
| Contract operation | The normalized operation as canonical JSON (sorted keys, no whitespace): method, path template, parameters with name, location, type, required; request body schema; responses by status with body schema; security requirements; events. Descriptions, examples, titles, and ordering are dropped |
| Acceptance criterion | Text with whitespace collapsed, case kept, list markers and trailing punctuation removed |
| Spec section | Text of the section under its heading, whitespace collapsed |
| Instruction file or page | Text with whitespace collapsed |
| Plan block, case | The block or the YAML loaded and dumped canonically, excluding `qa_meta` and the `fp` attribute |

`framework/qa/fingerprint.py` holds the one implementation, used by tools and connectors alike.

## 6.2 sources.lock

```yaml
plan_version: "1.2"
locked_at: 2026-11-14T22:10:00Z
framework_version: 1.4.0
spec:
  ref: {repo: "<repo>", path: specs/payments/spec.md}
  commit: 9c41e2a
  sections: {"refunds": 3d1f7e0c9b2a, "limits": 88a0c1d2e3f4}
contracts:
  - ref: {repo: "<repo>", path: specs/payments/contracts/api.yaml}
    version: 2.3.0
    operations:
      "POST /payments/authorize": {fingerprint: 5f1c9a0b7d21, api: authorize}
      "POST /refunds": {fingerprint: a77d02e41c90, api: refund}
stories:
  PAY-31:
    status: Ready for QA
    updated: 2026-11-13T16:40:00Z
    ac: {"PAY-31#AC1": e09a55b1c2d3, "PAY-31#AC2": 4d8b13a0f9e8}
instructions:
  "instructions/service.md": {fingerprint: 1a90ce3b4d5e}
  "page:123456": {version: "14", fingerprint: d4472f0a1b2c}
```

Only versions and fingerprints, never content (AI-14). Absent `sources.lock` means every source is new.

## 6.3 Classifying contract changes

`detect-changes` diffs the normalized operations from the lock's session cache (the previous snapshot is rebuilt from the specs connector at the locked commit) against the current ones.

| Change | Class |
|---|---|
| New operation | `new_api` if it belongs to no known API, else `additive` |
| Operation removed | `removed` |
| Method or path template changed | `breaking` (old removed, new added) |
| Request: new required parameter or field, field removed, type narrowed, enum value removed, required flag added | `breaking` |
| Request: new optional parameter or field, enum value added, required flag removed | `additive` |
| Response: field removed, type changed, field made optional, enum value added, success status code changed or removed | `breaking` |
| Response: new field, new error status code, enum value removed | `additive` |
| Security requirement changed | `breaking` |
| Event added | `additive` |
| Event removed, or its payload changed in a breaking way as above | `breaking` |
| Only descriptions or examples changed | No change (not in the fingerprint) |

Response enum additions are breaking because a client, including a test, can receive a value it does not expect. When the contract cannot be normalized well enough to classify a change, the change is `breaking` (safe default) and the plan notes it.

Criterion and instruction changes:

| Change | Class |
|---|---|
| Criterion added | `ac_added` |
| Criterion text fingerprint changed | `ac_changed` |
| Criterion gone from the story | `ac_removed` |
| Story left the tracked statuses | `story_on_hold` ([6.6](#66-ticket-statuses)) |
| Spec section fingerprint changed | `spec_changed` for each behavior that cites the section |
| Instruction fingerprint changed | `instruction_changed` for each section and case that lists it in `influenced_by` or `traces_to.instructions` |

## 6.4 Version numbers

| Version | Form | Bumped by |
|---|---|---|
| Section | `MAJOR.MINOR` per API section, starting at `1.0` | Major: `breaking` or `removed`. Minor: any other change to that section |
| Plan | `MAJOR.MINOR` per service | Minor: any section changed, a gap opened or closed. Major: template migration, or `qa plan --major` |
| Framework | Semantic version | Releases of the framework repo |

Every merged plan version gets the tag `<service>/plan-v<MAJOR.MINOR>`. Tagging happens on merge through the `ci_cd` connector in Phase 2; in Phase 1 the person who merges runs `qa tag <service>`. No `v1` or `v2` copies of any file exist (CV-8).

Two plan generator runs on unchanged inputs produce no diff (NF-4): `version` bumps only when `detect-changes` found something and the written blocks differ from what was there.

## 6.5 Stale cases and hand-edited cases

Staleness is computed, never stored, so it cannot drift. A case is **stale** when any of these differ from the current state:

- `traces_to.plan.section_version` major number differs from the section's current major number.
- Any `traces_to.ac[].fingerprint` or `traces_to.spec[].fingerprint` differs from the current source.
- An instruction in `traces_to.instructions` changed fingerprint since the case's `qa_meta.generated_by.session`.

`qa status` lists stale cases; `read-plan` feeds them to the case generator; the engine adds `stale: true` to their results so the analysis can say a failure may come from an outdated case.

Hand-edited cases (`qa_meta.edited_by_person: true`, or a file whose fingerprint no longer matches `qa_meta.generated_fp`):

1. The generator never writes the file, apart from setting `edited_by_person: true`.
2. If the case is stale, the generator writes the case it would have produced, computes a three-way diff (last generated version from git history, the person's version, the new version), and puts it in the PR description under "Proposed changes to edited cases" with the reason.
3. If the behavior was removed, the case is listed for retirement in the PR; the person sets `status: retired`.
4. A person accepts a proposal by applying it and setting `edited_by_person: false`, which hands the case back to the generator.

## 6.6 Ticket statuses

- The statuses that count are `sources.tracker.statuses` in `agent.yaml`, defaulting to `defaults.tracker_statuses` in the firm profile.
- Stories in other statuses are ignored when first seen.
- A story that was tracked and moves back out of the list is `on_hold`: its behaviors and cases keep their files, the plan section notes it, and the case generator marks cases `blocked` with `blocked_by: [on_hold:<key>]`. They come back when the story returns to a tracked status.
- Only a criterion removed from the story, or a story deleted or closed as not done, retires behaviors (TP-11).

## 6.7 Deployed version and pending cases

- The engine asks `deployed_version.get(service, env)` once per run and records the answer in `run.json`.
- Versions compare as semantic versions; a version with a build suffix compares by its numeric part.
- A case with `applies_from_service_version` greater than the deployed version is `pending_deploy` (CV-7).
- When the version is unknown (no endpoint, an error, or an unparseable value), the engine runs every case, records `deployed_service: unknown`, and adds `version_unknown: true` to the run. The executor's analysis treats a failure on a case newer than the last known deployed version as possibly `not deployed` and says so. `--assume-version` overrides the lookup and is recorded.

## 6.8 Side-by-side API versions

When a contract carries two major versions of the same API (for example operations under `/v1/refunds` and `/v2/refunds`):

- Each major version is its own API folder: `apis/refund` for the first one seen, `apis/refund-v2` for the next, with `version_of: refund` in `agent.yaml` and in the plan section.
- Each has its own plan section, cases, data template, and data files. Data values are not shared, because the versions may need different inputs; the plan generator notes which variables match so people can copy values.
- Removing `/v1` from the contract retires the `refund` section and cases as with any removed API.
- Which operation belongs to which API is decided by `scan-layout` from the path prefix or the contract's own version grouping, and written to `agent.yaml` `apis.<api>.operations`, where a person can correct it.

## 6.9 Workflows

For the later workflow phase. Each workflow folder holds `workflow.yaml` (the use case steps), `workflow.lock`, `test-plan.md`, `cases/`, and `data/`.

`workflow.lock` records, for every service API a workflow step calls: the service, API, section version, and operation fingerprint at the time the workflow plan was built. When any of them changes, the workflow steps that call it are stale, the same as a service case. Workflow cases reuse the service's environment data files by reference (`${payments.refund:captured_payment_id}`) instead of copying values (WF-4).
