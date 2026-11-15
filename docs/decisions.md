# Decisions

"Owner" marks a decision made by the product owner. "Default" marks the recommended default, adopted because the question was left open; any default can be revisited.

| # | Topic | Decision | By |
|---|---|---|---|
| D1 | Agents | Four agents: plan generator, case generator, executor (with summary and analysis), optimizer and learner; plus an on-demand validator generator | Owner |
| D2 | Workflows | Workflow agents in a later phase, same pattern; one plan and one set of cases per workflow | Owner |
| D3 | Results | Execution results and logs kept separate from code and test assets, outside git | Owner |
| D4 | Plan granularity | One plan per service with a section per API; can move to per API later. Plans and cases tagged with tickets. The generator detects existing services and folders and creates or updates them | Owner |
| D5 | Versions | Git history, a version number in each plan, a change log, and a tag per merged version | Default |
| D6 | Data files | YAML; a case may list several named data sets | Default |
| D7 | Secrets | Vault; data files hold references only | Owner |
| D8 | Executor | The agent selects what to run and calls a deterministic engine, then summarizes and analyzes | Default |
| D9 | Approvals | None mandatory. Review is a team process. Every step can run manually or on demand; later the whole pipeline runs in CI/CD | Owner |
| D10 | Optimizer and learner | Updates its learning record directly; all other changes as pull requests | Default |
| D11 | Test management | Results only, pushed to the test management tool; git stays the source of truth for cases. An optional plugin can push cases one way | Default + Owner |
| D12 | Interfaces | REST first, then gRPC, events, databases | Default |
| D13 | Test assets location | One central repo (or one per domain) | Default |
| D14 | Results store | Object storage with a run index now; results database later | Default |
| D15 | Validators | Catalog in the framework; validator generator is its own on-demand agent | Default |
| D16 | Wiki pages | Read live through a connector; no copies; version and fingerprint recorded | Owner |
| D17 | Steps and plugins | Every step runs by default; flags turn steps and plugins off or on; the case generator supports plugins | Owner |
| D18 | Portability | No firm, product, or project names in the framework; one firm profile selects a connector per concern | Owner |
| D19 | Repository | One repository with `framework/` and `test-assets/` to start; split later. Engine in Python | Default |
| D20 | Step kinds | Deterministic steps run as Python tools; only reasoning steps call the agent runtime. Both modes (headless and interactive) use the same step contracts. See [design 1.2](design/01-system-model.md#12-deterministic-and-reasoning-steps) | Default |
| D21 | Plan format | Markdown with `qa:generated`, `qa:shared`, and `qa:person` markers and one machine-readable `yaml qa-api` block per API section. See [design 2.5](design/02-artifacts.md#25-test-plan-servicesservicetest-planmd) | Default |
| D22 | Case status | `active`, `blocked`, `quarantined`, `obsolete`; no approval states. Hand edits detected by fingerprint. See [design 2.6](design/02-artifacts.md#26-test-case-apisapicasescase-idyaml) | Default |
| D23 | Engine failures | One `failed` status with an engine-observed `failure_kind`; failure classes stay in the analysis. See [design 3.7](design/03-engine.md#37-verdict-rules) | Default |
| D24 | Version numbers | Plan and section versions are both `MAJOR.MINOR`; breaking API changes bump the section major only. See [design 6.4](design/06-versioning.md#64-version-numbers) | Default |
| D25 | Fingerprints | 12 hex characters of SHA-256 over a canonical form that ignores descriptions and examples; staleness computed, never stored. See [design 6.1](design/06-versioning.md#61-fingerprints) | Default |
| D26 | Unknown deployed version | Run every case and flag the run; `--assume-version` overrides. See [design 6.7](design/06-versioning.md#67-deployed-version-and-pending-cases) | Default |
| D27 | Side-by-side API versions | Each major API version is its own API folder linked by `version_of`. See [design 6.8](design/06-versioning.md#68-side-by-side-api-versions) | Default |
| D28 | Run index | One index object per run so parallel writers never collide. See [design 8.2](design/08-results-store.md#82-run-index) | Default |
| D29 | Change policy | Generators never overwrite. No change in sources writes nothing. Existing cases keep their test content; a case that is no longer valid is marked `obsolete` and tagged `obsolete` and `obsolete:<reason>`, and a new case with a new id covers the changed requirement. No duplicate cases. See [design 6.5](design/06-versioning.md#65-change-policy-obsolete-and-add-never-overwrite) | Owner |
