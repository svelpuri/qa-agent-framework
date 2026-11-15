# test-assets

Per-service test content, created and maintained by the agents and by people. Results and logs are not stored here; they go to the results store. Secret values are never stored here; data files reference the vault.

| Path | Holds |
|---|---|
| `firm-profile.yaml` | The connector chosen for each concern |
| `registry/` | Which services and environments exist |
| `instructions/` | Person-owned global instructions |
| `services/<service>/` | Service context, `sources.lock`, test plan, change log, learnings, instructions, plugin state |
| `services/<service>/apis/<api>/` | Test cases, data template, environment data files, API instructions, generated plugin outputs |
| `workflows/<workflow>/` | Later phase: one plan and one set of cases per use case |

See [../docs/folder-structure.md](../docs/folder-structure.md) for every file.
