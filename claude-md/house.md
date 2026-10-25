# house

What Toghen's repositories share. Toghen is a design-tokens toolkit split into one repository per package: lib, postcss, query, ui, figma and skills. House is the seventh. It holds no package.

The rules are in the constitution, at `.specify/memory/constitution.md`, where house's own speckit reads them. It is the only copy in house; every other repository copies it from there. This file holds only what is not a rule.

## What house holds

| Path | What it is |
|---|---|
| `.specify/memory/constitution.md` | The shared constitution: the shared principles, one section per package, and its governance |
| `claude-md/` | The `CLAUDE.md` each repository carries, one file per repository |
| `speckit-preset/` | The house preset for speckit: the spec, plan and tasks templates and the analyze command, in the house register |
| `.reference/plans/workspace/` | The plans of the rounds that cross repositories |

Still to come: the Biome config, the purity check, the CI file and the local workspace files.

## How a repository copies from house

- **The constitution** goes to the repository's `.specify/memory/constitution.md`. Its first line is the one the constitution's governance names, with the house version.
- **Its `CLAUDE.md`** comes from `claude-md/<name>.md`, copied to the repository's root as `CLAUDE.md`.
- **The preset** is installed with `specify preset add --dev`, from a checkout of house.
- **The shared files** (the Biome config, the purity check, the CI file) are copied whole, each with a line naming the house version it came from.
- **The local workspace files** are copied by a command into the folder that holds the checkouts. That folder is not a repository.

A change to anything shared is made here first, then copied out.

## Commands

House has no build and no tests yet. `pnpm check` is coming, with the Biome config and the purity check.
