# 1. System model

## 1.1 Components and boundaries

| Component | Lives in | Language | Calls a model? | Decides pass or fail? |
|---|---|---|---|---|
| `qa` CLI and step runner | `framework/qa/` | Python | No | No |
| Tools (`scan_layout`, `detect_changes`, `validate_artifacts`, `new_service`, `check_connector`) | `framework/qa/tools/` | Python | No | No |
| Engine | `framework/engine/` (package `qa_engine`) | Python | No | **Yes** |
| Validators | `framework/validators/` | Python | No | **Yes**, through the engine |
| Connectors | `framework/connectors/<concern>/<name>/` | Python | Only `agent_runtime` | No |
| Plugins | `framework/plugins/<name>/` | Python | No | No |
| Guards | `framework/hooks/` | Python | No | Block only |
| Agent skills | `framework/skills/<agent>/SKILL.md` | Markdown instructions | **Yes** | No |

Two rules follow from the table and are enforced by tests in the framework repo:

- Nothing under `engine/`, `validators/`, `tools/`, `hooks/`, or `plugins/` imports the `agent_runtime` connector.
- Agents never write results. They read `results.json` and write `report.md`, recommendations, and learnings.

The `qa` package is installed once per machine or CI image (`pip install ./framework`). The engine is a separate distribution (`qa-engine`) so it can run in an execution environment without the agent tooling.

## 1.2 Deterministic and reasoning steps

Every step is one of two kinds.

| Kind | Run by | Examples |
|---|---|---|
| `tool` | The step runner, in Python | `scan-layout`, `read-sources`, `detect-changes`, `version`, `validate`, `commit`, `select-cases`, `check-data`, `run`, `publish`, `query-runs`, `open-pr`, `plugins` |
| `agent` | The agent runtime, through the `agent_runtime` connector, with the agent's skill and a step prompt | `write-plan`, `design-cases`, `write-cases`, `data-template`, `analyze`, `report`, `update-learnings`, `recommend` |

`data-template` is an agent step only for descriptions and examples; the variable list itself is computed by a tool from the cases, so it can never miss a variable (see [4.2](04-agents.md#42-test-case-generator)).

Splitting steps this way keeps flags, ordering, skipped-step records, validation, and commits identical whether a person runs an agent interactively or a pipeline runs it headless. The model only does the parts that need judgment.

## 1.3 Two ways to run an agent

| Mode | Started by | Who drives the steps |
|---|---|---|
| **Headless** | `qa plan payments` in a shell, an event, or CI/CD | The step runner. For each `agent` step it calls `agent_runtime.run_step(...)` and waits for the step's output files |
| **Interactive** | A person invokes the skill in the agent runtime, for example `/qa:plan payments` | The agent session. The skill tells the agent to call `qa session start`, then `qa step <name>` for each `tool` step, do each `agent` step itself, and call `qa session finish` |

Both modes produce the same work folder, the same session record, and the same outputs. Tests compare them on the sample service.

## 1.4 The `qa` CLI

| Command | Purpose |
|---|---|
| `qa plan <service> [--api <api>]` | Test Plan Generator |
| `qa cases <service> [--api <api>]` | Test Case Generator |
| `qa run <service> --env <env> [--api <api>] [--case <id>] [--tag <tag>]` | Test Executor |
| `qa optimize <service> [--runs <n>]` | Optimizer and Learner |
| `qa validator <service>:<gap-id>` | Validator Generator |
| `qa steps <agent> <service> [flags]` | Print the resolved step list for a run without running it |
| `qa step <agent> <step> --session <id>` | Run one `tool` step inside an interactive session |
| `qa session start <agent> <service> [flags]` / `qa session finish --session <id>` | Open and close an interactive session |
| `qa status <service>` | Plan version, stale cases, blocked cases, open gaps, missing environment values |
| `qa new-service <service>` | Scaffold a service (also called by `scan-layout`) |
| `qa validate [<path>]` | Check artifacts against schemas and traceability rules |
| `qa check-connector <concern> <name>` | Run the connector conformance kit |
| `qa package-runtime --out <dir>` | Build the runtime-specific plugin through the `agent_runtime` connector |
| `qa-engine run --service <s> --env <e> [...]` | The engine on its own; the executor's `run` step calls this |

Common flags on the four agent commands and `qa validator`:

| Flag | Effect |
|---|---|
| `--skip <step>[,<step>]` | Turn steps off for this run |
| `--only <step>[,<step>]` | Run only these steps |
| `--plugin <name>=on\|off` | Turn a plugin on or off for this run (repeatable) |
| `--dry-run` | Run every step but write nothing outside the work folder; `commit`, `publish`, and `open-pr` print what they would do |
| `--json` | Print the headless result object ([1.7](#17-headless-output-and-exit-codes)) instead of text |
| `--budget-usd <n>` | Lower the cost cap for this run ([7.5](07-plugins-guards.md#75-cost-control-and-the-off-switch)); cannot raise it |
| `--assets <path>` | Test assets repo root; default: walk up from the current folder to the first `firm-profile.yaml` |

## 1.5 Step resolution

The step runner builds the step list for one run in this order:

1. Start from `framework/config/agents.yaml`: the ordered steps for the agent, all on.
2. Apply the service's `agent.yaml` `steps.<agent>.skip` list, then its `plugins.<agent>` settings.
3. Apply `--only`: every step not listed is off. Then apply `--skip`. Then apply `--plugin` flags.
4. Reject unknown step or plugin names with exit code 2, listing the valid names. A typo never silently runs everything.
5. `--only` and `--skip` naming the same step is an error.

Order never changes; flags only switch steps on or off. `qa steps` prints the result with the source of each decision, for example `publish: off (agent.yaml)`.

### Missing inputs from skipped steps (SF-6)

Each step declares its inputs as files produced by earlier steps. When a needed input's producer is off:

| Situation | Behavior |
|---|---|
| The last saved output of that step exists for this service in `.local/state/<agent>/<service>/<step>.json`, and the step declares `reuse_last: allowed` | Use it, and record `{"step": "<consumer>", "used_last_saved": "<producer>", "saved_at": "..."}` in the session record |
| Otherwise | Stop before the consumer runs, exit code 3, message: `write-plan needs detect-changes output; run with detect-changes on, or run it once to save an output` |

`reuse_last` is declared per step in `agents.yaml`. It is `allowed` for `scan-layout`, `read-sources`, `detect-changes`, `select-cases`, and `query-runs`, and `forbidden` for anything that depends on the current working tree, such as `validate` output feeding `commit`.

## 1.6 Work folders and sessions

Every run of an agent is a **session** with id `ses-<yyyymmddThhmmZ>-<4 hex>`.

```
<test-assets>/.local/
  work/<session-id>/              one folder per session; kept 14 days, then removed by `qa` on start
    session.json                  resolved steps, flags, trigger, start and end, cost
    <step>.in.json                inputs handed to an agent step (paths and parameters, never secret values)
    <step>.out.json               outputs of each step
    logs/<step>.log
  state/<agent>/<service>/<step>.json   last saved output per step, for SF-6
```

`.local/` is already ignored by git. Session records also go to the results store under `sessions/` ([8.1](08-results-store.md#81-layout)) so audits (NF-5) do not depend on one machine.

`session.json` fields:

| Field | Meaning |
|---|---|
| `session_id`, `agent`, `service`, `api` | What ran |
| `trigger` | `{type: manual\|event\|ci, by: <user or system>, event: <event name or null>, ref: <commit, ticket, or deploy id>}` |
| `framework_version`, `test_assets_commit` | Versions used |
| `steps` | Ordered list: `{name, kind, state: ran\|skipped\|failed\|stopped, decided_by: default\|agent.yaml\|flag, used_last_saved, started, finished}` |
| `plugins` | `{name, enabled, decided_by, outcome: ok\|failed\|skipped, message}` |
| `cost` | `{usd, input_tokens, output_tokens, cap_usd}` |
| `outputs` | Branch, PR link, run id, files written |
| `result` | `ok`, `no_changes`, `stopped`, or `failed`, plus a message |

## 1.7 Headless output and exit codes

With `--json`, every agent command prints one object:

```json
{
  "agent": "plan-generator",
  "service": "payments",
  "session_id": "ses-20261115T1830Z-9f2c",
  "result": "ok",
  "headline": "payments plan 1.2 to 1.3: refund changed (additive), 2 cases stale, 1 new validation gap",
  "skipped_steps": ["commit"],
  "outputs": {"branch": null, "pr": null, "run_id": null},
  "details": {}
}
```

`details` is agent-specific and listed per agent in [chapter 4](04-agents.md).

| Exit code | Meaning |
|---|---|
| 0 | Finished. For `qa run`, also every non-quarantined case passed or was pending |
| 1 | `qa run` only: at least one non-quarantined case failed |
| 2 | Usage error: unknown step, plugin, service, or environment |
| 3 | Stopped before generating: a source unreachable, an input missing, a guard refused |
| 4 | Validation failed: an artifact does not match its schema or traceability rules |
| 5 | Cost cap reached or agents switched off |
| 6 | Internal error |

CI/CD reads exit codes 0 and 1 for gating. Codes 3 to 6 mean the run says nothing about the product.

## 1.8 Triggers

Only the trigger changes between phases (TR-4). The `ci_cd` connector turns each event into the same `qa` command a person would type.

| Event | Fired when | Command |
|---|---|---|
| `story.changed` | A story in a tracked status is created, changes status into the tracked list, or its acceptance criteria change | `qa plan <service>` then `qa cases <service>` |
| `spec.merged` | A commit that touches the spec or contract paths in `agent.yaml` lands on the service repo's main branch | `qa plan <service>` then `qa cases <service>` |
| `deploy.completed` | A non-production environment reports a new deployed version | `qa run <service> --env <env>` |
| `schedule.regression` | A schedule in the CI/CD system fires | `qa run <service> --env <env>` then, weekly, `qa optimize <service>` |

Event payloads carry the service, the environment where relevant, and a reference (commit, ticket, deploy id) that goes into `session.trigger.ref`. Flags for event-driven runs come from an optional `triggers:` block in `agent.yaml`, for example `deploy.completed: {flags: "--skip publish"}`.
