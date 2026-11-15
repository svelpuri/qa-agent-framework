# 7. Plugins, guards, and security

## 7.1 Plugins

A plugin adds an output to an agent without changing it (PL-2). Each plugin is a folder `framework/plugins/<name>/` with a manifest and a Python module.

`plugin.yaml` (PL-3):

```yaml
name: tracker-cases
version: 1.0.0
agents: [case-generator]              # any agent may host plugins (PL-7)
produces: One test item in the tracker or test management tool per case, created or updated
writes_to: external                   # external | services/<svc>/apis/<api>/generated/<name>/
state: services/<svc>/plugins/tracker-cases.map.yaml
connector: test_management            # concern it needs, or none
inputs: [cases]                       # cases | data-template | plan | run | report
config_schema:
  type: object
  properties: {project: {type: string}}
timeout_s: 300
```

Interface:

```python
@dataclass(frozen=True)
class PluginContext:
    service: str; agent: str; session_id: str
    config: dict                       # from agent.yaml plugins.<agent>.<name>
    cases: list[TestCase]              # new and obsolete cases in this session, plus all cases on --full
    assets_root: str
    output_dir: str | None             # for writes_to under generated/
    state: dict                        # loaded from the state file
    connector: Any | None
    dry_run: bool

@dataclass
class PluginResult:
    ok: bool; message: str
    written: list[str]; external_ids: dict[str, str]
    state: dict                        # saved back to the state file by the step runner

def run(ctx: PluginContext) -> PluginResult: ...
```

Rules:

- The `plugins` step runs enabled plugins in name order, each in its own process with its timeout. An exception, a timeout, or `ok: false` is recorded in `session.json` and the next plugin runs (PL-6).
- Outputs are generated one way from the cases, matched by case id through the state file, so a second run updates instead of duplicating (PL-4).
- Files under `generated/` start with a header naming the plugin, the case id, and "do not edit". `validate` rejects changes to them made outside the plugin.
- A plugin that writes externally must honor `dry_run` by reporting what it would do.

Phase 1 ships one example plugin, `case-index`, that writes `generated/case-index.md` per API (a readable table of cases). It needs no connector and exercises the whole mechanism. `tracker-cases` and `browser-tests` follow in Phase 2 with their connectors.

## 7.2 Guards

Guards are runtime-neutral Python functions in `framework/hooks/guards.py`. The `agent_runtime` connector wires them into the runtime's hook system when it packages the framework, and the step runner and engine call them directly.

| Guard | Signature | Blocks |
|---|---|---|
| Non-production | `check_url(url, environment) -> Decision` | Any host not in the environment's `allowed_hosts`, any environment with `production: true`, and in agent sessions any URL to a host in a production environment listed in the registry |
| Person-owned files | `check_write(path, content_before, content_after) -> Decision` | Writes to any `instructions/` folder, `apis/<api>/data/<env>.yaml`, `firm-profile.yaml`, `registry/environments.yaml`; changes to `qa:person` blocks or to person-owned `qa:shared` blocks in a plan; changes to any existing case outside the fields allowed in [6.5](06-versioning.md#65-change-policy-obsolete-and-add-never-overwrite) |
| Write scope | `check_scope(path, allowed_paths) -> Decision` | Writes outside the service folder and work folder of the current session |
| Secret leak | `check_text(text) -> Decision` | Text written by an agent that contains a registered secret value or a `${secret:...}` reference outside an environment data file |

`Decision` is `allow` or `block` with a reason. A block in an agent session returns the reason to the agent; a block in a tool step stops the step with exit code 3. `validate` repeats the person-owned checks against the base branch, so a guard that a runtime failed to wire is still caught before commit.

## 7.3 Secrets

- Values live only in the vault and in engine memory during a run (D7, ST-8).
- Data files hold `${secret:<path>}`; the path convention is `<env>/<service>/<key>`.
- The engine is the only caller of `secrets.resolve`; agents never see values. `check-data` reports a missing secret by path, never by value.
- The masker ([3.8](03-engine.md#38-masking)) applies to everything the engine writes.

## 7.4 Access to the results store

Logs hold request data (NF-3). The `results_store` connector is configured with a writer identity for the engine and executor and a reader identity for the optimizer and for people. Retention per environment comes from the firm profile (ST-5); the connector applies it with the store's lifecycle rules, or with `qa results prune` for the local folder store.

## 7.5 Cost control and the off switch

| Control | Where | Effect |
|---|---|---|
| `agents_enabled: false` | Firm profile | Every agent command exits 5 before any step. Tool-only commands (`qa-engine run`, `qa validate`, `qa status`) still work |
| `QA_AGENTS_DISABLED=1` | Environment variable | Same, for one machine or pipeline |
| `cost.cap_usd_per_session` | Firm profile; `budget_usd` in `agent.yaml` and `--budget-usd` can only lower it | The step runner passes the remaining budget to each agent step; when spent, the step stops, the session ends with exit 5, and outputs so far stay in the work folder, uncommitted |
| `cost.cap_usd_per_day` | Firm profile | Checked at session start against session records for the day in the results store |

## 7.6 Audit

Every session writes `session.json` ([1.6](01-system-model.md#16-work-folders-and-sessions)) to the results store: trigger and who started it, framework and test-asset versions, resolved steps and what decided each, inputs by path and version, outputs, cost, the runtime's own session reference, and the commit or PR it produced (NF-5). Every run record links to the session that started it.
