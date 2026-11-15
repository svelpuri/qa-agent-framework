# 5. Connectors

Every external system is reached through a connector with a fixed interface (PT-2). The firm profile picks one connector per concern (PT-3). Product-specific code lives only inside a connector implementation; nothing else in the framework imports one directly.

## 5.1 Packaging and loading

```
framework/connectors/
  base.py                      shared types and errors
  <concern>/
    interface.py               the Protocol below
    conformance.py             checks every implementation must pass
    <name>/
      connector.yaml
      impl.py
```

`connector.yaml`:

```yaml
name: local-files
concern: requirements
version: 1.0.0
implements: requirements/1          # interface name and major version
config_schema:                      # JSON Schema for `config:` in the firm profile
  type: object
  properties: {root: {type: string}}
credentials:                        # read from the process environment, never from files in git
  - {env: QA_TRACKER_TOKEN, required: false}
```

The loader reads the firm profile, finds `connectors/<concern>/<name>/`, checks `implements` against the interface major version the framework expects, validates `config`, and constructs `impl.Connector(config, credentials)`. A connector outside the framework (a firm's own) is loaded from any folder listed in `connector_paths` in the firm profile, with the same rules.

## 5.2 Shared types and errors

```python
class ConnectorError(Exception): ...
class Unreachable(ConnectorError): ...     # network, timeout, service down: agents stop (AI-15)
class NotFound(ConnectorError): ...
class Unauthorized(ConnectorError): ...
class Conflict(ConnectorError): ...         # write rejected because the target changed
class Unsupported(ConnectorError): ...      # optional operation not provided

@dataclass(frozen=True)
class Versioned:
    version: str          # the source's own version: commit, page version, update time
    fingerprint: str      # see 6.1
```

Every method is synchronous, has a timeout from the firm profile (`timeout_s`, default 30), and raises only these errors. Optional operations raise `Unsupported`, and callers handle that by recording the gap, not crashing.

## 5.3 Interfaces

### requirements

```python
class Requirements(Protocol):
    def list_stories(self, scope: TrackerScope) -> list[StoryRef]: ...
    def get_story(self, key: str) -> Story: ...
    def subscribe(self, scope: TrackerScope, callback_url: str) -> str: ...   # optional, Phase 2

@dataclass(frozen=True)
class Story:
    key: str; title: str; status: str; updated: str; url: str
    acceptance_criteria: list[Criterion]       # empty when the story has none (PT-6)
    labels: list[str]; epic: str | None

@dataclass(frozen=True)
class Criterion:
    id: str            # "<key>#AC<n>"; the connector derives n from the source's own order or ids
    text: str
    fingerprint: str
```

`TrackerScope` is the `sources.tracker` block from `agent.yaml`. The connector, not the agent, decides how criteria are found in the tracker (a field, a checklist, a heading), and must keep criterion ids stable when one criterion is edited.

### specs

```python
class Specs(Protocol):
    def get_spec(self, ref: SourceRef) -> Document: ...                     # text, sections by anchor
    def get_contract(self, ref: SourceRef) -> Contract: ...
    def changed_since(self, ref: SourceRef, version: str) -> bool: ...

@dataclass(frozen=True)
class Contract:
    version: str; format: str                  # format: the contract's description language
    operations: dict[str, Operation]           # key "METHOD /path" or "Service/Method"

@dataclass(frozen=True)
class Operation:
    key: str
    normalized: dict        # request, responses, auth, events: the shape used for fingerprints and diffs
    fingerprint: str
```

The connector normalizes the contract into `Operation.normalized` ([6.1](06-versioning.md#61-fingerprints)), so change detection never parses a contract format itself.

### instructions

```python
class Instructions(Protocol):
    def read(self, page: str) -> Page: ...             # live read, every time (AI-13)

@dataclass(frozen=True)
class Page:
    id: str; title: str; url: str
    version: str; fingerprint: str
    text: str             # held in memory and in .local/work only for the session; never committed
```

Instruction files in the test assets repo are read directly from disk and do not need a connector.

### secrets

```python
class Secrets(Protocol):
    def resolve(self, path: str, environment: str) -> str: ...
    def exists(self, path: str, environment: str) -> bool: ...
```

`resolve` refuses any path whose environment segment is marked `production: true` in the registry, as a second guard. Values never pass through the agent runtime: only the engine calls `resolve`.

### results_store

```python
class ResultsStore(Protocol):
    def write_run(self, run: RunFolder) -> str: ...                  # returns the run's location
    def write_index(self, entry: IndexEntry) -> None: ...
    def query(self, q: RunQuery) -> list[IndexEntry]: ...
    def read(self, location: str, path: str) -> bytes: ...
    def write_session(self, record: dict) -> None: ...
    def write_optimizer(self, service: str, name: str, body: bytes) -> str: ...
    def link(self, location: str) -> str: ...                        # a URL a person can open
```

`RunQuery` fields: `service`, `env`, `api`, `since`, `until`, `plan_version`, `limit`. See [chapter 8](08-results-store.md).

### test_management

```python
class TestManagement(Protocol):
    def push_results(self, run: RunSummary, results: list[CaseResult]) -> str: ...
    def upsert_case(self, case: TestCase, external_id: str | None) -> str: ...   # optional, used by a plugin
    def obsolete_case(self, external_id: str, reason: str) -> None: ...          # optional
```

Git stays the source of truth (D11). `upsert_case` exists only for the plugin that pushes cases one way.

### ci_cd

```python
class CiCd(Protocol):
    def to_command(self, event: Event) -> list[str]: ...    # maps an event to a qa command line (1.8)
    def report_status(self, session: dict) -> None: ...
    def open_pr(self, branch: str, title: str, body: str) -> str: ...
    def comment(self, pr: str, body: str) -> None: ...      # optional
```

Branch creation and commits use git directly; `open_pr` is the only call that needs the code host, so it lives here.

### traces

```python
class Traces(Protocol):
    header_name: str                                          # e.g. "traceparent"
    def new_trace_id(self) -> str: ...
    def spans(self, trace_id: str) -> list[Span]: ...         # optional; used by analysis and workflows
```

### service_auth

```python
class ServiceAuth(Protocol):
    def credentials(self, profile: str, environment: str) -> Credentials: ...

@dataclass(frozen=True)
class Credentials:
    headers: dict[str, str]          # registered with the masker before use
    client_cert: tuple[str, str] | None
    expires_at: str | None
```

Cases use `${auth:<profile>}` in a header; profiles are defined in `agent.yaml` under `auth_profiles`.

### deployed_version

```python
class DeployedVersion(Protocol):
    def get(self, service: str, environment: str) -> str | None: ...   # None when unknown
```

The default connector `version-endpoint` calls `base_url + version_endpoint` from the registry and reads the field named in its config.

### agent_runtime

```python
class AgentRuntime(Protocol):
    def package(self, framework_root: str, out_dir: str) -> None: ...
    def run_step(self, req: StepRequest) -> StepResult: ...
    def capabilities(self) -> set[str]: ...      # e.g. {"skills", "hooks", "subagents", "headless"}

@dataclass(frozen=True)
class StepRequest:
    agent: str; step: str; session_id: str
    workdir: str                 # the session folder; inputs in <step>.in.json
    skill_path: str
    allowed_paths: list[str]     # write scope; enforced by guards inside the runtime
    budget_usd: float

@dataclass(frozen=True)
class StepResult:
    ok: bool; message: str
    outputs: list[str]           # files written
    cost_usd: float; input_tokens: int; output_tokens: int
    session_ref: str             # the runtime's own id, for audit
```

`package` turns `runtime/manifest.json`, `skills/`, `AGENT_RULES.md`, and the guards in `hooks/` into the runtime's own plugin format, with hook wiring that calls the runtime-neutral guard functions ([7.2](07-plugins-guards.md#72-guards)) (PT-5). This is the only connector allowed to call a model.

## 5.4 Conformance kit

`qa check-connector <concern> <name>` runs `connectors/<concern>/conformance.py` against the implementation with the firm's real config (PT-4). Every kit checks:

- It loads, its config validates, and declared credentials are present.
- Each required method returns the declared types on a known input the firm supplies in `conformance:` config (for example a story key, a page id, a secret path that holds a dummy value).
- Fingerprints are stable: two reads of an unchanged source give the same fingerprint.
- Errors map correctly: an unknown id raises `NotFound`; a bad credential raises `Unauthorized`.
- Concern-specific checks: `secrets` refuses a production path; `results_store` round-trips a run and finds it by query; `instructions` reads live (the version changes after the firm edits the test page during the check, when run with `--live-edit`); `agent_runtime` runs a no-op step within budget and reports cost.

## 5.5 Local connectors for Phase 1

| Concern | Connector | Behavior |
|---|---|---|
| requirements | `local-files` | Stories as YAML files under a folder; criteria are a list |
| specs | `git-files` | Spec and contract files from a local or remote git repo at a ref; works for real firms too |
| instructions | `local-files` | Pages as Markdown files with a version in front matter |
| secrets | `local-env` | Values from environment variables named from the path; refuses production paths |
| results_store | `local-folder` | The layout in chapter 8 on disk |
| test_management | `none` | Accepts and logs; pushes nothing |
| ci_cd | `local-git` | Commits and branches locally; `open_pr` prints the PR body |
| traces | `w3c` | Generates trace ids; no span lookup |
| service_auth | `static` | Headers from secret references |
| deployed_version | `version-endpoint` | As above |
| agent_runtime | One connector for the runtime the team uses first | Chosen when Phase 1 starts |
