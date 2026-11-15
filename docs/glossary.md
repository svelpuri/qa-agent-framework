# Glossary

| Term | Meaning |
|---|---|
| **Acceptance criterion (AC)** | A testable condition in a user story; a primary source of expected results |
| **Agent** | A reasoning component that reads, writes, and analyzes; never decides pass or fail |
| **Agent runtime** | The environment that runs agents with tools, hooks, plugins, and a headless mode; chosen per firm |
| **Connector** | Code that reaches one external system through a fixed interface |
| **Data file** | Real values for one environment (`data/<env>.yaml`); secrets appear as vault references |
| **Data template** | Generated list of every variable an API's cases need (`data/template.yaml`) |
| **Engine** | The deterministic executor: binds variables, calls services, applies validators, records results |
| **Firm profile** | One file that selects a connector per concern for a firm |
| **Fingerprint** | A short hash of a source's content, used to detect change without storing a copy |
| **Instructions** | Person-owned guidance beyond stories and the spec, as files or wiki pages, at global, service, API, plan, or case level |
| **Learning record** | `learnings.md`: what the learner has observed across runs, each entry citing its runs |
| **Oracle** | The source of an expected result; must be a requirement or contract, never the implementation |
| **Pending deploy** | A case written for a service version not yet deployed; reported, not failed |
| **Plugin** | An optional step that adds an output, configured by manifest |
| **Run index** | One entry per run in the results store, for queries across runs |
| **sources.lock** | The versions and fingerprints of every source a plan was built from |
| **Stale case** | A case whose recorded source fingerprints no longer match the current sources |
| **Step** | A named part of an agent's work; all on by default, switchable by flag |
| **Validation** | One thing a test must check |
| **Validation gap** | A required validation with no validator in the catalog; blocks the affected cases |
| **Validator** | A deterministic, reusable check, listed in the catalog |
| **Workflow** | A use case that chains API calls across services; later phase |
