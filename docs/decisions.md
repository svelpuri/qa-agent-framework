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
