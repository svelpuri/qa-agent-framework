# Agents

All agents are generic. Each service runs its own instance, given that service's folder as context. Every step runs by default; flags change that for one run.

Common flags:

| Flag | Effect |
|---|---|
| `--skip <step>[,<step>]` | Turn steps off for this run |
| `--only <step>[,<step>]` | Run only these steps |
| `--plugin <name>=on\|off` | Turn a plugin on or off for this run |
| `--dry-run` | Report what would change; write nothing |

Precedence: framework defaults, then the service's `agent.yaml`, then flags. Skipped steps are always listed in the output.

---

## Test Plan Generator

Creates or updates the test plan for one service.

**Command:** `qa plan <service> [--api <api>]`

**Reads:** `agent.yaml`; the spec and API contracts; user stories and acceptance criteria (only the statuses configured for the service); instruction files and wiki pages, read live; the current plan, `CHANGELOG.md`, `sources.lock`, `learnings.md`; the validator catalog.

**Steps**

| Step | What it does |
|---|---|
| `scan-layout` | Compares registry, folders, and contract APIs; creates missing service and API folders; reports orphans |
| `read-sources` | Builds a snapshot of every source with versions and fingerprints; stops if any source is unreachable |
| `detect-changes` | Compares the snapshot with `sources.lock`: added, changed, removed APIs and acceptance criteria; changed instructions; conflicts |
| `write-plan` | Updates only affected API sections: requirements, ticket tags, behaviors, test types, priority, data needs, validations, gaps |
| `version` | Bumps plan and section versions, writes the change log entry and the new `sources.lock` |
| `validate` | Checks the plan against its schema and traceability rules |
| `commit` | Commits to a branch and opens a pull request |

**Never:** writes to instruction files or person-written plan sections; decides between conflicting sources; generates from partial inputs.

---

## Test Case Generator

Writes test cases and data file templates from the latest plan.

**Command:** `qa cases <service> [--api <api>]`

**Reads:** the plan including person-written instructions; instruction files and wiki pages; the validator catalog; `learnings.md`.

**Steps**

| Step | What it does |
|---|---|
| `read-plan` | Loads the plan and finds new APIs and stale cases |
| `design-cases` | Chooses cases per behavior: functional, negative, boundary, contract, event, data |
| `write-cases` | One YAML file per case, tagged with its tickets; every input is a `${variable}`; each check names a catalog validator, or the case is blocked by a gap |
| `data-template` | Regenerates the API's data template with every variable; flags new ones for people to fill per environment |
| `validate` | Checks schemas, traceability, and that only catalog validators are used |
| `plugins` | Runs enabled plugins, for example pushing cases to the issue tracker or exporting browser tests |
| `commit` | Commits to a branch and opens a pull request |

**Never:** overwrites a case marked `edited_by_person: true` (proposes the change instead); derives expected results from the implementation.

---

## Test Executor

Runs test cases in one environment, then summarizes and analyzes the run.

**Command:** `qa run <service> --env <env> [--api <api>] [--case <id>]`

**Steps**

| Step | What it does |
|---|---|
| `select-cases` | Picks cases by service, API, id, or tag; skips retired cases |
| `check-data` | Resolves every variable from the environment data file and vault references; lists all missing values before any call |
| `run` | The engine runs the cases, applies validators, and decides pass or fail; cases for an undeployed service version are reported as pending |
| `analyze` | Classifies failures: product, test, data, environment, dependency, requirement gap, flaky |
| `report` | Writes the run report: headline, counts, failures with evidence, what to do differently, versions used |
| `publish` | Sends results to the test management tool and a short summary with a link to the full run |

**Never:** changes a pass or fail result; runs against production; writes secrets to logs.

---

## Test Optimizer and Learner

Learns across runs and proposes improvements.

**Command:** `qa optimize <service> [--runs <n>]`

**Steps**

| Step | What it does |
|---|---|
| `query-runs` | Reads recent runs from the run index, grouped by plan version per API |
| `analyze` | Finds flaky, redundant, slow, or failing-for-data cases, coverage gaps, unclear instructions |
| `update-learnings` | Updates `learnings.md` directly; every entry cites its runs; contradicted entries are removed |
| `recommend` | Writes recommendations: evidence, expected benefit, exact change |
| `open-pr` | Opens a pull request with the proposed changes; never merges |

**Never:** changes expected results or pass and fail rules through learnings.

---

## Validator Generator (on demand)

Closes a validation gap listed in a test plan.

**Command:** `qa validator <gap-id>`

Writes a deterministic validator and its tests, adds a catalog entry, and opens a pull request. Cases blocked by the gap are unblocked once the validator passes its tests and is merged.
