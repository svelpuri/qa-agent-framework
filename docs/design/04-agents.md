# 4. Agents and step contracts

Each step lists its kind (`tool` or `agent`, see [1.2](01-system-model.md#12-deterministic-and-reasoning-steps)), what it reads, what it writes to the work folder, and when it stops. "Stops" means exit code 3 with a message naming the cause; nothing after it runs.

Every agent step receives the same base context in `<step>.in.json`: the service, the session id, paths to `agent.yaml`, `learnings.md`, and the applicable instruction sources in precedence order (case, plan, API, service, global; AI-8). Agents read files directly; the step runner never pastes page content into the prompt, so wiki pages stay live (AI-13).

## 4.1 Test Plan Generator

| Step | Kind | Reads | Writes | Stops when |
|---|---|---|---|---|
| `scan-layout` | tool | Registry, folders, contract operations | Creates missing service and API folders (`qa new-service`), API entries in `agent.yaml`; `scan-layout.out.json`: created, orphans | Service unknown and `--dry-run` is off and the name is invalid |
| `read-sources` | tool | Spec, contracts, stories, instruction files and pages through connectors | `read-sources.out.json`: the new snapshot (versions and fingerprints only) plus local cache paths for this session | Any source is unreachable (AI-15); the message names it |
| `detect-changes` | tool | Snapshot, `sources.lock` | `detect-changes.out.json`: classified changes per API and per criterion ([6.3](06-versioning.md#63-classifying-contract-changes)) | Never. "No changes" ends the session with `result: no_changes` |
| `write-plan` | agent | Changes, current plan, sources, instructions, learnings, catalog | Updated `test-plan.md` generated and shared blocks for affected APIs only | The agent's output fails the plan schema twice |
| `version` | tool | Changes, old and new plan | Section and plan versions, `CHANGELOG.md` entry, new `sources.lock` | Never |
| `validate` | tool | Everything changed | `validate.out.json` | Any error (exit 4) |
| `commit` | tool | Working tree | Branch `qa/plan/<service>/v<version>`, PR with the headline, counts, conflicts, and proposals for person-owned blocks | The repo has unrelated uncommitted changes in the service folder |

`write-plan` rules for the agent (in the skill, checked where possible by `validate`):

- Touch only API sections listed in `detect-changes.out.json`, plus sections 4 and 5.
- Every behavior lists at least one source (TP-13). Every validation lists its behaviors and either validators or a gap.
- Conflicting sources become a requirement gap of kind `conflict`. The agent does not pick a side (AI-10).
- Record `influenced_by` for every instruction used (AI-11).

`details` in headless output: `{from_version, to_version, apis: {added, changed, removed, on_hold}, stale_cases, validation_gaps: {opened, closed}, requirement_gaps, branch, pr}`.

## 4.2 Test Case Generator

| Step | Kind | Reads | Writes | Stops when |
|---|---|---|---|---|
| `read-plan` | tool | Plan, cases, `sources.lock`, catalog | `read-plan.out.json`: per API, behaviors without cases, invalid cases with their reason, gaps now closed | The plan fails its schema. Nothing to do ends the session with `result: no_changes` and writes nothing |
| `design-cases` | agent | `read-plan` output, plan, instructions, learnings | `design-cases.out.json`: one entry per planned case: behavior, type, priority, validators, data needs, the ids it replaces; entries whose duplicate key matches an active case are dropped, and invalid cases whose test would not change are marked confirmed ([6.5](06-versioning.md#65-change-policy-obsolete-and-add-never-overwrite)) | Never |
| `write-cases` | agent | The design | New case files only. Existing cases get only the allowed field changes: `obsolete` status, tags, and block for invalid cases | Never; schema errors are caught by `validate` |
| `data-template` | tool + agent | Cases | `template.yaml`: the tool computes the variable list, types, `used_by`, and `new_in_plan`; the agent fills only `description` and `example` | Never |
| `validate` | tool | Changed files | `validate.out.json` | Any error (exit 4) |
| `plugins` | tool | Enabled plugins for `case-generator` | Plugin outputs and state ([7.1](07-plugins-guards.md#71-plugins)) | Never; a failing plugin is reported (PL-6) |
| `commit` | tool | Working tree | Branch `qa/cases/<service>/<short>`, PR listing new cases, obsolete cases with their reason and replacement, confirmed cases, blocked cases, and new variables per environment; hand-edited cases that became obsolete are shown next to their replacement | As for the plan generator |

Rules enforced by `validate`:

- Every case traces to a plan version, section version, and at least one behavior (TC-9) and carries a `ticket:` tag per ticket (TC-10).
- Every request value that is not a constant from the contract (method, path template, header names) is a `${variable}` (TC-3). Literal hosts, ids, and amounts fail validation.
- Every validator is an `active` catalog entry and its parameters match the schema (VL-8).
- Cases for a behavior blocked by a validation or requirement gap have `status: blocked` and `blocked_by`.
- Existing cases are unchanged except for the fields allowed in [6.5](06-versioning.md#65-change-policy-obsolete-and-add-never-overwrite), checked against the base commit.
- No two active cases share a duplicate key.

`details`: `{plan_version, cases: {new, obsolete, confirmed, blocked}, new_variables: {<env>: [<names>]}, plugins: [{name, outcome}], branch, pr}`.

## 4.3 Test design method

Rules for `design-cases`, in the skill and in `framework/docs/methodology.md`:

| Coverage item | Cases |
|---|---|
| Each behavior | At least one case of the behavior's first test type |
| Each acceptance criterion | Exactly one positive case, unless the criterion lists several outcomes |
| Each error code in the spec or contract for the operation | One negative case |
| Each numeric or length limit | Boundary cases at the limit and one step past it on each side that changes the outcome |
| Each operation | One `contract` validation (`json_schema`) on a success response, attached to an existing positive case rather than a new case |
| Each event the spec says the operation emits | One case that checks it is emitted on success and one that checks it is absent on failure |
| Each operation that requires authorization | One `auth` case with a missing or invalid credential |
| Each instruction that asks for a scenario | One case, with the instruction in `traces_to.instructions` |

Priority comes from the plan section, raised to P1 for auth and for cases on a behavior marked critical in instructions. Learnings may change data choices, waits, and ordering; they never change an expected result (LN-5).

## 4.4 Oracle rules

From `framework/docs/oracle-rules.md`:

1. An expected result MUST come from an acceptance criterion, the spec, or the contract. Implementation code, existing responses, and previous run results are never oracles (TC-7).
2. For behavior (what happens), the order is: acceptance criterion, then spec. For shape (fields, types, codes), the contract.
3. When two sources disagree, the plan records a `conflict` requirement gap and blocks the behaviors it affects. Cases for those behaviors are `blocked` until a person resolves the gap by changing a source or writing a person block in the plan that settles it.
4. Instructions decide what and how to test, not what is correct. An instruction that states an expected result different from the sources is a conflict, handled as in rule 3.
5. When no source states an expected result for a behavior an instruction asks for, the case is `blocked` by a requirement gap of kind `missing_source`.

## 4.5 Test Executor

| Step | Kind | Reads | Writes | Stops when |
|---|---|---|---|---|
| `select-cases` | tool | Filters, case statuses | `select-cases.out.json`: case ids and the reason each was left out | No case matches (exit 2) |
| `check-data` | tool | Engine stages 1 to 7 in dry-run mode | `check-data.out.json`: missing variables per case and environment | Any value missing (exit 3, all listed) |
| `run` | tool | Selection | The run folder and index entry, through the engine | Engine stops (codes 3, 4, 6) |
| `analyze` | agent | `results.json`, logs, previous runs for failing cases (from the index), learnings | `analysis.json`: one entry per failure with class, confidence, evidence links, suggested action | Never |
| `report` | agent | Analysis, run record | `report.md` in the run folder | Never |
| `publish` | tool | Run, report | Results to the `test_management` connector; summaries to configured targets | Never; publishing errors are reported and the run stands |

`analyze` rules: classify each failure with one class from [4.7](#47-failure-taxonomy); cite at least one log line or result field per claim (EX-13); never change or contest a verdict (EX-12). When the evidence does not support a class, use `test` with `confidence: low` and say what evidence is missing.

`report.md` follows `templates/run-report.md`: one-sentence headline (the most important finding), counts table, failures table (case, step, validator, class, evidence link), what to do differently, and versions used. Every claim links to `logs/<case>.jsonl#L<n>` or a `results.json` entry.

`details`: `{run_id, env, counts: {passed, failed, pending_deploy, blocked, skipped}, quarantined_failed, headline, report_path}`.

## 4.6 Optimizer and Learner

| Step | Kind | Reads | Writes | Stops when |
|---|---|---|---|---|
| `query-runs` | tool | Run index (last `--runs`, default 30, per environment) | `query-runs.out.json`: per case: outcomes, durations, failure kinds, grouped by section version and deployed version | Fewer than 3 runs (result `no_changes`) |
| `analyze` | tool + agent | Query output, cases, plan, learnings | The tool computes the signals below; the agent explains them and proposes changes | Never |
| `update-learnings` | agent | Signals, existing learnings | `learnings.md` (direct write; D10) | Never |
| `recommend` | agent | Signals | `recommendations/R-<date>-<n>.md` in the results store `optimizer/` folder, and the file changes they propose | Never |
| `open-pr` | tool | Proposed changes | Branch `qa/optimize/<service>/<date>`, PR; never merges (OP-5) | Never |

Signals the tool computes (deterministic, so recommendations cite numbers):

| Signal | Definition |
|---|---|
| Flaky | Within the window, the same case, data set, section version, and deployed version both passed and failed |
| Slow | Case duration at the 90th percentile is more than twice the median of its API, and over 2 seconds |
| Redundant | Two active cases with the same operation, the same validator set with equal parameters, and requests that differ only in values from the same data set |
| Data problem | Repeated stops at `check-data`, or failures the executor classified as `data` in 3 or more runs |
| Coverage gap | A plan behavior with no active case, or a validation never exercised in the window |
| Unclear instruction | The same reviewer change to generated cases in 2 or more PRs (read from the PR history through the `ci_cd` connector, Phase 2) |

Learnings: the `yaml qa-learnings` block in `learnings.md` holds `{id, learning, applies_to, evidence: [run ids], since, last_confirmed}`. An entry is removed when the last 5 runs that could confirm it contradict it, or when it has not been confirmed in 60 days (LN-4). Learnings never mention expected values or validator parameters that set pass or fail; `validate` rejects entries that reference a validator's `expected`, `max_ms`, or rule values (LN-5).

## 4.7 Failure taxonomy

From `framework/docs/failure-taxonomy.md`. The executor picks exactly one class per failed case.

| Class | Choose when | Evidence required |
|---|---|---|
| `product` | The service returned something the oracle forbids and the request was valid | The request, the response, and the source of the expected value |
| `test` | The case itself is wrong: wrong path, wrong validator parameter, a step that cannot succeed as written | The case line and the source showing the correct value |
| `data` | The environment data made the request invalid: an id that does not exist, an expired token, an amount that breaks a precondition | The data variable and the response showing the cause |
| `environment` | Transport failure, timeout, or a 5xx from infrastructure in front of the service | `failure_kind` and the transport error or gateway response |
| `dependency` | The service reported a failure in a service it calls | The response or trace span naming the dependency |
| `requirement_gap` | The service behaved consistently, but the sources are ambiguous or disagree about whether that behavior is correct | The two sources, or the missing source |
| `flaky` | The case has a flaky signal or learning and nothing in this run points elsewhere | The learning id or the runs that show both outcomes |

## 4.8 Validator Generator

`qa validator <service>:<gap-id>` runs in the framework repo checkout named in `agent.yaml`'s pinned framework version.

| Step | Kind | Reads | Writes | Stops when |
|---|---|---|---|---|
| `read-gap` | tool | The plan's `qa-gaps` block, the catalog | `read-gap.out.json`: what must be checked, interfaces, behaviors | Gap not found or already closed |
| `design` | agent | Gap, catalog, nearby validators | `design.json`: name, kind, interface, parameter schema, test list | The gap is already covered by an active validator (the agent says which; the plan generator will map it next run) |
| `write` | agent | Design | `validators/<name>.py`, `validators/tests/test_<name>.py`, catalog entry with `status: proposed` | Never |
| `test` | tool | New files | Test results; runs the tests twice and compares outputs to check determinism | Any test fails after the agent's two fix attempts |
| `validate` | tool | Catalog | Schema and import checks (no model or network imports) | Any error |
| `open-pr` | tool | Working tree | PR in the framework repo; the PR body links the gap | Never |

Required tests per validator: at least one passing and one failing input per parameter, a malformed response, and missing optional parameters. Merging the PR and releasing the framework moves the entry to `status: active`; the next `qa plan` run closes the gap and the next `qa cases` run unblocks its cases.
