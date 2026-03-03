# Investigate Aletheia

## Purpose

Determine if Google DeepMind's Aletheia (autonomous mathematical research agent) can be recreated using Claude Code.

## First-Run Setup

On first use in a fresh clone, follow `docs/FIRST_RUN.md` to initialize project memory files.

## Environment

- **Dependencies**: Managed via `flake.nix`. Add packages to `buildInputs`.
- **Initialization**: `.envrc` sources `use flake` and runs scripts from `.envrc.d/` and `.envrc.local.d/`.
- If you add a dependency to `flake.nix`, ask the user to restart the session so `direnv` reloads.

## Workflow

<!-- Describe the steps Claude Code should follow when working on this project. -->

## Writing Standards

- Structured with headers, bullet points, and blockquotes for key statements.
- No filler or padding. Dense, scannable, useful.

## Lessons Learned

<!-- Add project-specific lessons as they arise. -->

## Contributing

All changes follow the workflow in [CONTRIBUTING.md](CONTRIBUTING.md). File a GitHub issue, use the bundled skills to brainstorm, plan, execute, verify, and review, then open a PR with the plan and issue reference.
