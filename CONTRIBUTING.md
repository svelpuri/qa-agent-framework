# Contributing

## Branches and pull requests

- Work on a branch; open a pull request to `main`.
- Agent-made changes use branches named `qa/<agent>/<service>/<short-description>`.
- Review follows the team's own process; nothing in the framework blocks on approval.

## Rules for the framework

- **No firm, product, or project names** in `framework/` or `docs/`. Use roles: issue tracker, wiki, vault, test management tool, CI/CD system, agent runtime. Product-specific code belongs only in a connector implementation.
- **Nothing service-specific** in `framework/`. Service facts belong in `test-assets/services/<service>/`.
- **Deterministic pass or fail.** Validators and the engine never call a model.
- **One fact, one place.** Link to the source of a fact instead of copying it.
- Every new artifact type gets a template in `framework/templates/` and a schema in `framework/schemas/`.

## Rules for test assets

- Never commit secret values; reference the vault.
- Do not edit files under `generated/`; they are rewritten by plugins.
- Instruction files and person-written plan sections are owned by people; agents read them only.

## Writing style for documents

- Plain, direct sentences; diagrams as SVG, not text art.
- No em dashes.
