# Detailed design

**Status:** draft for review. Follows the [requirements baseline](../requirements.md) (draft v11) and the [decisions](../decisions.md). Where this design picks a default for something the requirements left open, the choice is listed in [Resolved items](#resolved-items) and added to the decision log as a "Default" that can be revisited.

**In short.** A single Python package, `qa`, owns everything deterministic: the step runner that applies flags, the tools, the engine that decides pass or fail, the guards, and the connector and plugin loaders. Agents are prompts (skills) that run in the agent runtime and do only the reasoning steps: writing plan sections, designing and writing cases, analyzing runs, writing learnings and recommendations. Every step reads and writes typed files in a work folder, so the same step works when a person runs it, when an event triggers it, and in CI/CD. Every artifact has a schema and is checked before commit. Change tracking is driven by normalized fingerprints in `sources.lock`. Generators never overwrite: no change means no write, and a case that is no longer valid is marked obsolete and tagged while a new case covers the change.

## Chapters

| # | Chapter | Covers |
|---|---|---|
| 1 | [System model](01-system-model.md) | Components and process boundaries, the `qa` CLI, the step runner, flags and precedence, work folders, agent sessions, headless output, exit codes |
| 2 | [Artifacts and schemas](02-artifacts.md) | Identifiers, every file in the test assets repo, plan markers, the test case model, variables and data sets, data templates and environment files |
| 3 | [Engine](03-engine.md) | Run pipeline, binding, version gating, adapters, validator interface and catalog, verdict rules, masking, timeouts, concurrency |
| 4 | [Agents and step contracts](04-agents.md) | Each agent's steps with inputs, outputs, and failure behavior; test design method; oracle rules; failure taxonomy; optimizer rules |
| 5 | [Connectors](05-connectors.md) | Connector packaging and loading, the interface for each of the twelve concerns, error model, conformance kit, local connectors |
| 6 | [Versioning and change tracking](06-versioning.md) | Fingerprints, `sources.lock`, change classification, breaking changes, version numbers, stale cases, hand-edited cases, side-by-side API versions, workflows |
| 7 | [Plugins, guards, and security](07-plugins-guards.md) | Plugin manifest and interface, guards, secrets, cost caps, the off switch, audit |
| 8 | [Results store](08-results-store.md) | Run folder files, the run index, retention, publishing, queries for the optimizer |

New diagrams: [engine run pipeline](../diagrams/fig11-engine-run-pipeline.svg) and [case status decisions](../diagrams/fig12-case-status-decisions.svg).

## Conventions in this design

- **MUST** and **MUST NOT** are hard rules that a tool or guard enforces. **SHOULD** is guidance for agents that `validate` cannot fully check.
- File and field names in `code` are final unless a chapter marks them "proposed".
- Examples use a sample service called `payments` with APIs `authorize` and `refund`. They are illustrations, not part of the framework.
- Schemas live in `framework/schemas/` as JSON Schema (draft 2020-12). This design lists their fields; the schema files are the source of truth once written.

## Resolved items

Every item marked TBD in the requirements, the docs, or the sample structure, and where this design settles it.

| Item | Resolution | Where |
|---|---|---|
| Version numbering: per service, per API, or both | Both. Plan `MAJOR.MINOR` per service; section `MAJOR.MINOR` per API | [6.4](06-versioning.md#64-version-numbers) |
| What counts as a breaking change | Fixed rule list over the normalized contract diff | [6.3](06-versioning.md#63-classifying-contract-changes) |
| Which ticket statuses trigger an update | Per service list in `agent.yaml`, defaulting to the firm profile; stories that leave the list go on hold, not retired | [6.6](06-versioning.md#66-ticket-statuses) |
| How the deployed version is discovered | `deployed_version` connector; default reads a version endpoint per environment; unknown versions run and are flagged | [6.7](06-versioning.md#67-deployed-version-and-pending-cases) |
| Side-by-side API versions (`/v1`, `/v2`) | Each major API version is its own API folder linked by `version_of` | [6.8](06-versioning.md#68-side-by-side-api-versions) |
| Merging into hand-edited cases | Never merged or rewritten. An invalid case, edited or not, is marked obsolete and tagged, and a new case replaces it; no change means no write | [6.5](06-versioning.md#65-change-policy-obsolete-and-add-never-overwrite) |
| How workflows track API versions | `workflow.lock` pins section versions and fingerprints of every service API a workflow calls | [6.9](06-versioning.md#69-workflows) |
| Validator catalog format | Catalog entry schema with `kind`, `interface`, parameter schema, status, version | [3.6](03-engine.md#36-validators-and-the-catalog) |
| Validator generator details | Inputs, outputs, required tests, acceptance rules | [4.8](04-agents.md#48-validator-generator) |
| Connector interfaces (all twelve) | Python protocols with errors and conformance checks | [5](05-connectors.md) |
| Final folder and file names | Confirmed with three additions: `qa_meta` in cases, `.local/` work folders, `index/runs/` objects | [2](02-artifacts.md) |
| Final step names | Confirmed as listed in [agents](../agents.md); validator generator gets its own steps | [4](04-agents.md) |
| Trigger commands and events | `qa <agent>` commands and four named events | [1.8](01-system-model.md#18-triggers) |
| Failure taxonomy | Seven classes with decision rules and required evidence | [4.7](04-agents.md#47-failure-taxonomy) |
| Methodology (what to test) | Coverage rules per behavior | [4.3](04-agents.md#43-test-design-method) |
| Oracle rules | Source order and conflict handling | [4.4](04-agents.md#44-oracle-rules) |
| Case status values | `active`, `blocked`, `quarantined`, `obsolete`; the sample's `reviewed` is dropped because nothing is ever marked approved | [2.6](02-artifacts.md#26-test-case-apisapicasescase-idyaml) |
| Transport errors versus validator failures | One status, `failed`, with a `failure_kind` field; no new status | [3.7](03-engine.md#37-verdict-rules) |
| Run index under parallel writers | One index object per run instead of one shared file | [8.2](08-results-store.md#82-run-index) |
| Service maturity in the registry | Kept as an informational field only; no behavior depends on it | [2.3](02-artifacts.md#23-registry) |

## Open for the owner

Nothing blocks Phase 1. These defaults are worth a second look before the design is baselined:

1. **Unknown deployed version.** Default: run the cases and flag the run, rather than marking new cases pending. Safer for coverage, noisier for analysis.
2. **Plan major version.** Default: the plan's major number changes only on a template migration or by request. Breaking API changes bump the section's major number, not the plan's.
3. **Fingerprint length.** Default: 12 hex characters of SHA-256, instead of the 6 in the sample, to make collisions negligible across many services.
