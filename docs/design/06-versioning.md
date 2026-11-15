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

## 6.5 Change policy: obsolete and add, never overwrite

Owner decision D29. Applies to both generators.

**1. No change, no write.** The plan generator stops with `no_changes` when `detect-changes` finds nothing. The case generator stops with `no_changes` when `read-plan` finds no behavior without a case, no invalid case, and no closed gap. In both cases nothing is written: no file, no branch, no PR (NF-4).

**2. Only the delta is touched.** The plan generator updates only the API sections whose sources changed, and every other section stays byte-identical. Person-owned blocks are never touched. The plan is one living file, and its history is the git history plus `CHANGELOG.md`.

**3. A written case's test content is never changed by a generator.** Its steps, requests, validations, expected values, data sets, and `traces_to` stay exactly as written, whether a generator or a person wrote them. A generator may change only these fields on an existing case:

| Field | Change allowed |
|---|---|
| `status` | To `obsolete`, `blocked`, or back to `active` from `blocked` |
| `tags` | Add `obsolete` and `obsolete:<reason>` |
| `obsolete` | Add the block below |
| `blocked_by` | Set or clear |
| `qa_meta` | Set `edited_by_person`, `confirmed_in_plan` |

`validate` compares every existing case with the base commit and fails the session if any other field changed. The guard `check_write` enforces the same rule while the agent works.

**4. Invalid cases become obsolete and are tagged.** A case is invalid when:

| Reason tag | Cause |
|---|---|
| `obsolete:requirement-changed` | The fingerprint of an acceptance criterion it traces to changed |
| `obsolete:spec-changed` | The fingerprint of a spec section it traces to changed |
| `obsolete:contract-breaking` | Its section's major version changed because of a breaking contract change |
| `obsolete:behavior-removed` | The behavior it tests was removed from the plan |
| `obsolete:api-removed` | Its API was removed from the contract |

The case generator then marks it:

```yaml
status: obsolete
tags: [ticket:PAY-20, obsolete, "obsolete:requirement-changed"]
obsolete:
  since_plan: "1.1"
  reason: requirement-changed
  cause: "PAY-20#AC2"                 # the source that changed
  detail: "Decline limit changed from 500 to 1000"
  replaced_by: [PAY-AUTH-TC-0007]     # empty when the behavior was removed
```

Obsolete cases stay in the folder so history stays readable. They are never run, their ids are never reused, and `qa status` and the PR list them. Plugins move their external copies to an obsolete state (for example, the tracker item).

**5. Changes get new cases.** For each invalid case whose behavior still exists, and for each new behavior, the generator writes a new case with the next free id. A replacement carries `replaces: [<old id>]`. Before writing any new case, the generator checks for duplicates. The duplicate key is the operation, the behavior ids, the case type, the validators with their parameters, and the data set names. If an active case already has the same key, no new case is written.

**6. A change that does not alter the test.** When a source changed but the case the generator would write now is identical in test content to the existing case (same key and same expected values), for example a reworded criterion, the case stays active. Only `qa_meta.confirmed_in_plan` is set to the new plan version, so it is no longer reported as stale. No new case is written.

**7. Hand-edited cases follow the same rules.** If a case a person edited becomes invalid, it is marked obsolete and tagged in the same way, using only the fields above, and a new case is written. The PR shows the old case next to the new one, so the person can carry their edits over. Instruction changes alone never make a case obsolete. They are listed in the PR for a person to judge, because instructions guide what to test, not what is correct (rule 4 of [4.4](04-agents.md#44-oracle-rules)).

**8. Overwrite on request.** `--overwrite` on `qa plan` or `qa cases` replaces rules 1 to 6 for that run, within the scope set by `--api` or `--case` (the whole service when neither is given):

| | Default | With `--overwrite` |
|---|---|---|
| Nothing changed in the sources | Nothing written | Regenerated anyway |
| Plan | Only changed API sections updated | Every generated block in scope rewritten, and `qa:shared` blocks no person has edited |
| Existing generated case | Kept; obsolete and tagged when invalid, plus a new case | Rewritten in place with the same id |
| Case for a behavior that no longer exists | Obsolete and tagged | Obsolete and tagged (a case without a behavior cannot be rewritten) |

These never change, even with the flag: instruction files and pages, `qa:person` blocks and `qa:shared` blocks a person edited, environment data files, and cases with `edited_by_person: true`. Hand-edited cases still follow rule 7, so the flag never discards a person's work. A run that wants to replace one of those must first have a person hand it back (for example by setting `edited_by_person: false`).

Other rules for the flag:

- A rewrite that comes out identical is not written, so `--overwrite` on unchanged inputs can still end with `no_changes`.
- A rewritten case keeps its id, gets a new `qa_meta.generated_fp`, and records `qa_meta.overwritten: {session, previous_commit}`. Git keeps the previous version.
- A plan rewrite bumps the plan's minor version and adds a `CHANGELOG.md` entry that starts "Overwritten on request".
- `validate` relaxes the allowed-fields check from rule 3 only for cases in scope, and the session record and PR list every file that was overwritten.
- The flag is accepted on the command line and in CI/CD job definitions only, never as a default in `agent.yaml`, so overwriting is always a decision made for one run.

**Stale** is the short window between a source change and the next case generator run. A case is stale when rule 4 would make it obsolete but the generator has not run yet. `qa status` lists stale cases, and the engine adds `stale: true` to their results so the analysis can say a failure may come from an outdated case. Staleness is computed from fingerprints, never stored. For a case confirmed under rule 6, the comparison uses the fingerprints in `sources.lock` as of the plan version in `qa_meta.confirmed_in_plan` (read from git history), not the older ones in its `traces_to`.

## 6.6 Ticket statuses

- The statuses that count are `sources.tracker.statuses` in `agent.yaml`, defaulting to `defaults.tracker_statuses` in the firm profile.
- Stories in other statuses are ignored when first seen.
- A story that was tracked and moves back out of the list is `on_hold`: its behaviors and cases keep their files, the plan section notes it, and the case generator marks cases `blocked` with `blocked_by: [on_hold:<key>]`. They come back when the story returns to a tracked status.
- Only a criterion removed from the story, or a story deleted or closed as not done, retires behaviors (TP-11), and their cases become obsolete (`obsolete:behavior-removed`).

## 6.7 Deployed version and pending cases

- The engine asks `deployed_version.get(service, env)` once per run and records the answer in `run.json`.
- Versions compare as semantic versions; a version with a build suffix compares by its numeric part.
- A case with `applies_from_service_version` greater than the deployed version is `pending_deploy` (CV-7).
- When the version is unknown (no endpoint, an error, or an unparseable value), the engine runs every case, records `deployed_service: unknown`, and adds `version_unknown: true` to the run. The executor's analysis treats a failure on a case newer than the last known deployed version as possibly `not deployed` and says so. `--assume-version` overrides the lookup and is recorded.

## 6.8 Side-by-side API versions

When a contract carries two major versions of the same API (for example operations under `/v1/refunds` and `/v2/refunds`):

- Each major version is its own API folder: `apis/refund` for the first one seen, `apis/refund-v2` for the next, with `version_of: refund` in `agent.yaml` and in the plan section.
- Each has its own plan section, cases, data template, and data files. Data values are not shared, because the versions may need different inputs; the plan generator notes which variables match so people can copy values.
- Removing `/v1` from the contract retires the `refund` section and makes its cases obsolete (`obsolete:api-removed`), as with any removed API.
- Which operation belongs to which API is decided by `scan-layout` from the path prefix or the contract's own version grouping, and written to `agent.yaml` `apis.<api>.operations`, where a person can correct it.

## 6.9 Workflows

For the later workflow phase. Each workflow folder holds `workflow.yaml` (the use case steps), `workflow.lock`, `test-plan.md`, `cases/`, and `data/`.

`workflow.lock` records, for every service API a workflow step calls: the service, API, section version, and operation fingerprint at the time the workflow plan was built. When any of them changes, the workflow steps that call it are stale, the same as a service case. Workflow cases reuse the service's environment data files by reference (`${payments.refund:captured_payment_id}`) instead of copying values (WF-4).
