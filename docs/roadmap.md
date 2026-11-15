# Roadmap

## Now

- [x] Requirements baseline ([requirements.md](requirements.md))
- [x] Repository with documentation
- [x] Detailed design document: interfaces, schemas, step contracts, connector interfaces, and every item marked TBD ([design/](design/README.md))

## Phase 1: first working slice (manual triggers)

- [ ] Engine: variable binding with data sets, REST adapter, validators (status, error code, schema, field rules, latency, stored state), version gating, local results writer and run index
- [ ] Tools: `scan_layout`, `detect_changes` with `sources.lock`, `validate_artifacts`, `new_service`
- [ ] Steps and flags for every agent; skipped steps recorded
- [ ] Agents: plan generator, case generator, executor with analysis
- [ ] Plugin mechanism with one example plugin
- [ ] Local connectors: files for stories and instructions, a local secrets stand-in, a folder results store
- [ ] A mock service and an end-to-end example service in `test-assets/`
- [ ] Unit tests and an end-to-end test

## Phase 2: learning and events

- [ ] Optimizer and learner
- [ ] Validator generator
- [ ] Event triggers: story change, spec merged, deploy to a test environment
- [ ] Connectors: issue tracker, wiki (live read), vault, object storage, test management
- [ ] gRPC, event, and database interfaces

## Phase 3: CI/CD and workflows

- [ ] Whole pipeline in CI/CD on commits, requirement changes, and regression schedules
- [ ] Results database and dashboards
- [ ] Workflow agents: plan and cases per use case, one trace id end to end, failing hop pinpointed
- [ ] Split `framework/` and `test-assets/` into separate repositories
