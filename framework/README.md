# framework

Generic code shared by every service and every firm. Nothing here names a service, firm, or product.

| Folder | Purpose | Status |
|---|---|---|
| `runtime/` | Runtime-neutral manifest; turned into the chosen runtime's plugin format by the runtime connector | Planned |
| `skills/` | One folder per agent with its instructions | Planned |
| `config/` | Default steps per agent | Planned |
| `templates/` | Templates every agent output follows | Planned |
| `schemas/` | Machine-checkable shape of every artifact | Planned |
| `validators/` | Validator catalog, shared validators, and their tests | Planned |
| `engine/` | Deterministic execution engine (Python) | Planned |
| `connectors/` | One interface per external concern, plus implementations | Planned |
| `plugins/` | Optional agent outputs, each with a manifest | Planned |
| `tools/` | Layout scan, change detection, artifact validation, scaffolding, connector checks | Planned |
| `hooks/` | Guards: non-production only; person-owned files read-only for agents | Planned |

See [../docs/folder-structure.md](../docs/folder-structure.md) for what each file holds and [../docs/roadmap.md](../docs/roadmap.md) for build order.
