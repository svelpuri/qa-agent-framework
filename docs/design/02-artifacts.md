# 2. Artifacts and schemas

Every artifact below has a template in `framework/templates/` and a schema in `framework/schemas/`. `qa validate` checks them, and the `validate` step of each agent runs it on everything the session changed.

## 2.1 Identifiers

| Thing | Format | Example | Unique within |
|---|---|---|---|
| Service | `[a-z][a-z0-9-]{1,39}` | `payments` | Test assets repo |
| API | `[a-z][a-z0-9-]{1,39}`, optional `-v<n>` suffix for side-by-side versions | `refund`, `refund-v2` | Service |
| Case prefix | `[A-Z]{2,6}`, set once in `agent.yaml` as `case_prefix` | `PAY` | Repo |
| API key | `[A-Z0-9]{2,8}`, set per API in `agent.yaml` `apis.<api>.key` | `REF` | Service |
| Test case | `<case_prefix>-<api key>-TC-<4 digits>` | `PAY-REF-TC-0004` | Repo; never reused, even after the case is obsolete |
| Behavior | `<api>.B<n>` | `refund.B3` | Service; never reused |
| Validation | `<api>.V<n>` | `refund.V4` | Service |
| Validation gap | `VG-<n>`; referenced across services as `<service>:VG-<n>` | `payments:VG-1` | Service |
| Requirement gap | `RG-<n>` | `RG-2` | Service |
| Learning | `L-<n>` | `L-3` | Service |
| Recommendation | `R-<yyyymmdd>-<n>` | `R-20261115-2` | Service |
| Run | `run-<yyyymmddThhmmZ>-<4 hex>` | `run-20261115T1830Z-b41c` | Results store |
| Session | `ses-<yyyymmddThhmmZ>-<4 hex>` | `ses-20261115T1830Z-9f2c` | Results store |
| Branch | `qa/<agent>/<service>/<short-description>` | `qa/plan/payments/v1.3` | Repo |
| Plan tag | `<service>/plan-v<MAJOR.MINOR>` | `payments/plan-v1.3` | Repo |

New ids are the next free number found by scanning existing files, including obsolete ones. The `validate` step rejects a duplicate id.

## 2.2 Firm profile (`test-assets/firm-profile.yaml`)

One per test assets repo. Selects a connector per concern and sets firm-wide defaults. Never holds a credential.

```yaml
framework_version: ">=1.0,<2.0"           # versions this repo accepts
agents_enabled: true                       # the off switch (NF-6)
cost:
  cap_usd_per_session: 5.00
  cap_usd_per_day: 200.00
defaults:
  tracker_statuses: ["<ready status>", "<done status>"]
  retention_days: {qa: 90, staging: 180}
connectors:
  requirements:     {connector: <name>, config: {url: <tracker url>}}
  specs:            {connector: git-files}
  instructions:     {connector: <name>, config: {url: <wiki url>}}
  secrets:          {connector: <name>, config: {address: <vault address>}}
  results_store:    {connector: <name>, config: {bucket: <bucket>}}
  test_management:  {connector: <name>, config: {push: results}}
  ci_cd:            {connector: <name>}
  traces:           {connector: <name>}
  service_auth:     {connector: <name>}
  deployed_version: {connector: version-endpoint, config: {field: version}}
  agent_runtime:    {connector: <name>, config: {model_endpoint: <endpoint>}}
```

Credentials for connectors come from the process environment or the `secrets` connector, named in each connector's `connector.yaml` ([5.1](05-connectors.md#51-packaging-and-loading)).

## 2.3 Registry

`registry/services.yaml`:

```yaml
services:
  payments:
    path: services/payments
    owner: team-payments
    maturity: assisted        # informational only: draft | assisted | independent | workflow-ready
```

The framework version a service uses lives only in its `agent.yaml` (one fact, one place); the sample's duplicate in the registry is dropped.

`registry/environments.yaml`:

```yaml
environments:
  qa:
    production: false                       # MUST be false; the engine refuses true
    allowed_hosts: ["*.qa.example.internal"]
    timeouts: {request_s: 30, case_s: 300}
    retries: {transport: 1}                 # retries for connection errors only
    services:
      payments: {base_url: "https://payments.qa.example.internal", version_endpoint: /version}
```

## 2.4 Service context (`services/<service>/agent.yaml`)

| Field | Required | Meaning |
|---|---|---|
| `service`, `owner` | Yes | Name and owning team |
| `framework_version` | Yes | Exact framework version pinned for this service |
| `case_prefix` | Yes | Prefix for case ids |
| `apis.<api>` | Written by `scan-layout` | `{key, operations: [<METHOD path>], version_of: <api or null>}` |
| `sources.spec` | No (PT-6) | `{repo, path, ref}`; `ref` defaults to the main branch |
| `sources.contracts` | No (PT-6) | List of `{repo, path, ref, format}` |
| `sources.tracker` | No (PT-6) | `{project, epics, labels, statuses}`; `statuses` defaults to the firm profile |
| `instructions` | No | List of `{level, path}` or `{level, page}` with `api` or `case` when the level needs it |
| `interfaces` | Yes | Subset of `rest`, `grpc`, `events`, `database` |
| `environments` | Yes | Names from `registry/environments.yaml` |
| `steps.<agent>.skip` | No | Service defaults for skipped steps |
| `plugins.<agent>.<plugin>` | No | `{enabled, ...plugin config}` |
| `triggers.<event>.flags` | No | Flags added when an event starts the agent |
| `budget_usd` | No | Lower per-session cap for this service |

A missing source is not an error. The plan generator lists it under requirement gaps, for example "No acceptance criteria source: every behavior traces to the spec only" (PT-6).

## 2.5 Test plan (`services/<service>/test-plan.md`)

Markdown for people, with machine-readable parts that tools parse. Three marker types divide the file:

| Marker | Owner | Rule |
|---|---|---|
| `<!-- qa:generated id=<id> fp=<fingerprint> -->` ... `<!-- /qa:generated -->` | Plan generator | Rewritten freely by the generator |
| `<!-- qa:shared id=<id> fp=<fingerprint> -->` ... `<!-- /qa:shared -->` | Generator writes the first version; a person may edit | `fp` is the fingerprint of the content the generator last wrote. If the current content no longer matches `fp`, a person edited it and it becomes person-owned: the generator never rewrites it again and proposes changes in the PR description instead |
| `<!-- qa:person id=<id> -->` ... `<!-- /qa:person -->` | People | Never written by agents; the instruction guard and `validate` both check it |

Text outside any marker is treated as person-owned. `validate` compares every person-owned block with the base branch and fails if an agent changed one.

Front matter:

```yaml
service: payments
plan_version: "1.2"
template_version: 1
framework_version: 1.4.0
tickets: [PAY-20, PAY-31]
built_from: sources.lock
```

Sections, in order: 1 Service overview (generated), 2 Scope and out of scope (shared), 3 APIs (generated, one subsection per API), 4 Validation gaps (generated), 5 Requirement gaps and questions (generated), 6 Environments and entry and exit criteria (shared), 7 Additional information and instructions (person), 8 Change log (a link to `CHANGELOG.md`). Person blocks may also appear inside an API subsection with `id=api/<api>/notes`.

Each API subsection holds a short human summary and one fenced block with the info string `yaml qa-api`, which is the machine-readable source of truth for that section:

````markdown
### 3.2 refund
<!-- qa:generated id=api/refund fp=7b1e04c9aa21 -->
Full and partial refunds of captured payments, with idempotent replay.

```yaml qa-api
api: refund
section_version: "1.0"
status: active                       # active | on_hold | retired
operations: ["POST /refunds"]
tickets: [PAY-31]
applies_from_service_version: "2.4"
priority: P1
behaviors:
  - id: refund.B1
    text: Full refund of a captured payment succeeds with status REFUNDED
    sources: ["PAY-31#AC1"]
    test_types: [functional]
  - id: refund.B2
    text: A refund above the captured amount is rejected with REFUND_EXCEEDS_CAPTURE
    sources: ["PAY-31#AC2", "spec#refunds"]
    test_types: [negative, boundary]
validations:
  - {id: refund.V1, checks: "201 with status REFUNDED", validators: [http_status, field_rules], behaviors: [refund.B1]}
  - {id: refund.V4, checks: "Replay with the same key returns the same refund and no second row", gap: VG-1, behaviors: [refund.B3]}
data_needs:
  - {name: captured_payment_id, description: A captured payment in the environment}
  - {name: api_token, secret: true}
influenced_by: ["instructions/service.md", "page:123456"]
```
<!-- /qa:generated -->
````

Validation gaps (section 4) and requirement gaps (section 5) are also fenced `yaml qa-gaps` blocks:

```yaml qa-gaps
validation_gaps:
  - id: VG-1
    needs: Two identical requests with one idempotency key return the same resource and create exactly one record
    interface: [rest, database]
    blocks: [refund.V4]
    status: open                    # open | in_progress | closed
    closed_by: null                 # validator name once closed
requirement_gaps:
  - id: RG-1
    item: "PAY-31#AC2"
    finding: Spec allows partial refunds within 30 days; the criterion has no time limit
    kind: conflict                  # conflict | missing_source | untestable | ambiguous
    blocks: []                      # behaviors blocked until resolved
    owner: product owner
```

## 2.6 Test case (`apis/<api>/cases/<case-id>.yaml`)

| Field | Required | Meaning |
|---|---|---|
| `id`, `title`, `service`, `api` | Yes | Identity; title is one sentence stating the behavior |
| `traces_to.plan` | Yes | `{version, section, section_version, behaviors: [<behavior ids>]}` |
| `traces_to.tickets` | Yes | Ticket keys |
| `traces_to.ac` | When an AC exists | `[{id, fingerprint}]` |
| `traces_to.spec` | When the spec is a source | `["<path>#<anchor>"]` with `fingerprint` per entry |
| `traces_to.instructions` | When instructions influenced it | Paths or `page:<id>` (AI-11) |
| `tags` | Yes | Includes `ticket:<KEY>` per ticket; free tags allowed |
| `type` | Yes | `functional`, `negative`, `boundary`, `contract`, `event`, `data`, `auth` |
| `priority` | Yes | `P1`, `P2`, `P3` |
| `status` | Yes | `active`, `blocked`, `quarantined`, `obsolete` |
| `obsolete` | When obsolete | `{since_plan, reason, cause, detail, replaced_by}` ([6.5](06-versioning.md#65-change-policy-obsolete-and-add-never-overwrite)) |
| `replaces` | When it replaces obsolete cases | Ids of the cases it replaces |
| `blocked_by` | When blocked | `[VG-<n>]` or `[RG-<n>]` |
| `applies_from_service_version` | Yes | Semantic version the behavior ships in |
| `datasets` | No | Names of data sets; the steps run once per data set |
| `setup`, `steps`, `teardown` | `steps` yes | Ordered step lists; `teardown` runs even when a step fails |
| `qa_meta` | Yes, written by tools | `{generated_by: {agent, framework_version, session}, generated_fp, edited_by_person, confirmed_in_plan, overwritten}` |

`status` meaning:

| Status | Set by | Engine behavior |
|---|---|---|
| `active` | Case generator | Runs |
| `blocked` | Case generator, when a validator or requirement is missing | Not run; reported `blocked` with the gap id |
| `quarantined` | A person (the optimizer may propose it by PR) | Runs; its verdict is recorded but does not affect the exit code |
| `obsolete` | Case generator, when the requirement it tests changed or was removed; also tagged `obsolete` and `obsolete:<reason>` | Not run; kept on disk; listed by `qa status` |

`qa_meta.generated_fp` is the fingerprint of the case as the generator last wrote it (excluding `qa_meta`). If the file's current fingerprint differs, a person edited it: tools treat it as `edited_by_person: true` whether or not the flag was set, and set the flag on the next generator commit. Generators never change the test content of any existing case, edited or not; the few fields they may change are listed in [6.5](06-versioning.md#65-change-policy-obsolete-and-add-never-overwrite).

### Steps

```yaml
steps:
  - name: refund_full
    interface: rest                 # default: rest
    request:
      method: POST
      path: /refunds
      headers: {Authorization: "Bearer ${api_token}", Idempotency-Key: "${idempotency_key}"}
      body: {payment_id: "${captured_payment_id}", amount: "${captured_amount}"}
    save: {refund_id: body.id}
    validate:
      - {validator: http_status, with: {expected: 201}, validation: refund.V1}
      - {validator: field_rules, with: {rules: [{field: body.status, equals: REFUNDED}]}, validation: refund.V1}
```

- `save` stores values from the response for later steps. Paths use the field path syntax below.
- Each `validate` entry names a catalog validator, its parameters, and the plan validation it implements, so a run result traces back to the plan (NF-1).

**Field paths.** `status`, `headers.<name>` (case-insensitive), `body`, `body.<key>`, `body.<key>[<index>]`, `body.<key>[*].<key>` (all items). No other expressions. Validators receive resolved values, never expressions.

## 2.7 Variables and data sets

| Form | Resolves to | Allowed in |
|---|---|---|
| `${name}` | A value from the active data set, then the environment data file, then a saved value from an earlier step | Cases |
| `${secret:<path>}` | A value from the `secrets` connector | Environment data files only; `validate` rejects it in a case |
| `${uuid}` | A new UUID per case execution | Cases and data files |
| `${now}` | Run start time, ISO 8601 UTC | Cases and data files |
| `${run_id}` | The current run id | Cases and data files |
| `$${` | A literal `${` | Anywhere |

- A saved value and a data value with the same name is a `validate` error.
- When a whole YAML value is one variable, the result keeps its type (`amount: "${captured_amount}"` becomes the number 80). Inside a longer string it is converted to text.
- Built-ins are resolved once per case execution, so the same `${uuid}` within one case is one value.

**Data sets.** A case with `datasets: [small, full]` runs once per name. Values come from `datasets.<name>` in the environment data file, falling back to `values`. Results are reported per case and data set as `PAY-REF-TC-0001[small]`.

## 2.8 Data template (`apis/<api>/data/template.yaml`)

Generated; never edited by people.

```yaml
api: refund
generated_from_plan: "1.2"
variables:
  captured_payment_id:
    type: string                    # string | number | integer | boolean | list | object | secret
    required: true
    description: A payment already captured in this environment
    example: pay_qa_cap_01          # never a real secret; omitted for type secret
    used_by: [PAY-REF-TC-0001, PAY-REF-TC-0002]
    new_in_plan: null               # plan version that first needed it, while any environment lacks a value
datasets: [small, full]             # data set names any case uses
```

## 2.9 Environment data file (`apis/<api>/data/<env>.yaml`)

Owned by people.

```yaml
environment: qa
api: refund
values:
  captured_payment_id: pay_qa_cap_01
  captured_amount: 80
  idempotency_key: ${uuid}
  api_token: ${secret:qa/payments/api_token}
datasets:
  small: {captured_amount: 10}
  full: {captured_amount: 80}
```

`<env>.local.yaml` next to it overrides values for one person's machine and is ignored by git. The engine logs that an override file was used, never its values.

## 2.10 Other files in a service folder

| File | Format | Written by |
|---|---|---|
| `sources.lock` | YAML, schema in [6.2](06-versioning.md#62-sourceslock) | Plan generator `version` step |
| `CHANGELOG.md` | Markdown; one `## <version> (<date>)` heading per version with bullets: what changed, which source caused it, affected APIs, PR link | Plan generator `version` step |
| `learnings.md` | Markdown table plus one fenced `yaml qa-learnings` block that tools read ([4.6](04-agents.md#46-optimizer-and-learner)) | Optimizer and learner |
| `instructions/**` | Free Markdown | People only |
| `plugins/<plugin>.map.yaml` | Plugin-defined YAML | The plugin |
| `apis/<api>/generated/**` | Plugin output with a do-not-edit header | The plugin |
