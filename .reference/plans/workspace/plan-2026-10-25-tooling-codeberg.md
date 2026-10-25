# Plan: the tooling round and the move to Codeberg (25 October 2026)

The first plank of the piece-by-piece rebuild agreed on 25 October: give the workspace the gates it lacks, then move the repository to a Codeberg organisation, history intact, still one repository. Splitting into several repositories is not part of this round. The organisation is named so that a split later reads naturally (question 7). Every finding below was checked on `main` at lib 6.0.0, postcss 0.21.0, query 0.2.0, figma 7.0.1, ui 0.3.2.

## 1. Findings

### 1.1 No linter, no formatter, no CI

The only gates are `pnpm typecheck`, `pnpm test` and `pnpm build`, run by hand. `toghen-figma` runs its typecheck before its tests (`pretest`). Nothing runs on a push. The repository holds no linter or formatter config and no workflow folder. The whole test run takes about 37 seconds on this Mac, so CI would be cheap.

The code is consistent anyway. All 375 source files indent with spaces; only `toghen-postcss/tsconfig.json` uses tabs. A formatter would change little, which matters for question 1.

### 1.2 The tests of lib and postcss are never type-checked

`toghen-lib/tsconfig.json` includes `src/**/*.ts` only, and `toghen-postcss/tsconfig.json` excludes `test`. The tests run through `tsx`, which strips types without checking them. So about 12,600 lines of tests compile in nobody's checker. Checking them as they are gives:

- lib, 6 errors. Two in `test/unit/core/design-system.test.ts` (lines 88 and 96) call a function with 1 argument where it takes 2 to 4. Those tests may not test what they say. The other four are loose union types and an implicit `any`.
- postcss, 2 errors, in `test/example.test.ts:260` and `test/fixes.test.ts:11`.

`toghen-query`, `toghen-ui` and `toghen-figma` already check their tests.

Unchecked on purpose, and left alone: the plugin's screen (`toghen-figma/ui/`, about 4,300 lines of plain JavaScript; the reason is in `tsconfig.plugin.json`) and `toghen-ui/src` (about 650 lines). Checking them is a separate decision.

### 1.3 The settings have drifted apart

- `engines`: lib claims Node 16 and later, query Node 18 and later, the rest say nothing. Nothing tests either claim. Both versions are long out of support, and the tests' own runner (`node --import tsx/esm --test`) needs a newer Node anyway.
- No `packageManager` field and no `.node-version` file. This Mac runs Node 25.6.1 and pnpm 10.29.3.
- Compile targets: ES2017 (lib, and the plugin's sandbox, which is deliberate) and ES2022 (everything else).

### 1.4 The purity rule has no check

`CLAUDE.md` calls the purity rule non-negotiable, and a person enforces it. A check that greps for the consumer's name would have to hold the name, and a list in the repository would break the rule's letter. Question 2.

The one tracked file that names the consumer is `CLAUDE.md` itself, where it explains the loop. That matters if the repository goes public (question 3).

### 1.5 The repository and what points at it

- GitHub, **private**, 6.3 MB, 625 commits since 29 June 2026, no open issues or PRs.
- 20 tracked files mention GitHub. The ones that point at Toghen's own repository:
  - the `repository` field of all six `package.json` files, plus `homepage` and `bugs` in lib's;
  - `toghen-lib/README.md` and `toghen-postcss/README.md` (their links to the docs);
  - `toghen-lib/guide.md:18`, installing from the repository by git URL;
  - `toghen-skills/plugins/toghen/.claude-plugin/plugin.json` (`repository`);
  - `toghen-skills/plugins/toghen/skills/figma/SKILL.md:26`, a link to `.reference/figma/measured_behaviour.md`.

  The other links point at third-party projects (FigCraft, KDL, mdn-data) and stay.
- The plugin marketplace is this repository's root (`.claude-plugin/marketplace.json`, the source a relative path). Wherever it is added by URL, it has to be added again from the new one.
- The downstream prototype installs from the repository by git URL and files issues on GitHub in batches. Each round starts from `gh issue list`. Both change.

### 1.6 Publishing

lib, postcss and query are on npm; figma and ui are private. They are published by hand from this Mac after a bump commit. npm's trusted publishing (publishing from CI without a stored token) accepts only GitHub Actions, GitLab CI/CD and CircleCI, not Forgejo or Woodpecker. So publishing from Codeberg's CI would need an npm token stored as a CI secret. Publishing stays manual in this round. The npm pages show the new repository only after the next publish of each package.

### 1.7 CI on Codeberg

Codeberg's docs say its hosted Forgejo Actions are limited, and they point people who need hosted CI to Woodpecker CI (`ci.codeberg.org`, which needs an application). The other way is a runner of one's own: a program on a machine that connects out to Codeberg and runs the jobs. It needs no public address, but it runs the workflow's code on that machine. Question 6.

### 1.8 What a public repository would show

Codeberg exists for free software, and Toghen is MIT and already on npm, so the natural form there is public (question 3). A public repository would show:

- the consumer's name, in `CLAUDE.md` (one paragraph);
- the keys of the two Figma test files, in `.reference/figma/measured_behaviour.md` and one plan. Anyone with a key can open the file if its sharing allows it;
- `egildo@tagliareni.it`, already public on npm;
- `.reference/` as a whole: plans, research, measurements. It reads as a lab notebook, which is a strength in public, not a leak.

No token or secret turned up in the history (searched for npm tokens). Removing something from *history*, not just from `main`, means rewriting it before the first public push.

## 2. Design

### 2.1 The gates (on `main`, before the move)

1. **Tests type-checked.** lib and postcss get a test config like figma's (`tsconfig.test.json`), and `typecheck` runs it. The 8 errors are fixed. A wrong-arity call is fixed by making the test test what its name says.
2. **One floor.** `engines.node` at the chosen floor (question 8) in every package, `packageManager: "pnpm@10.29.3"` at the root, and a `.node-version` file.
3. **Linter and formatter** (question 1). Config at the root. Lint findings are measured first. A rule that fires many times in working code is switched off with a one-line reason, rather than mass-rewriting the plugin in a tooling round. Behaviour does not change in this round. The reformat is one commit of its own, listed in `.git-blame-ignore-revs`.
4. **`pnpm check`** at the root: lint, typecheck, test, build, then the leak check `CLAUDE.md` asks for by hand, now a script. The leak check fails if the built plugin contains `[object Object]` or an unresolved `toghen(`. CI runs exactly this, so local and CI agree.
5. **The purity check**, if question 2 says so.

### 2.2 The move (after the gates are merged)

1. Egildo creates the organisation on Codeberg (account actions are his). It holds one repository (question 7).
2. The history goes over by the method question 4 picks.
3. One commit rewrites the URLs listed in 1.5. Then patch bumps of lib, postcss and query and a publish, so npm points to Codeberg.
4. GitHub is archived or mirrored (question 5). The local `origin` changes, and the marketplace is added again from the new URL.
5. The memories that say "GitHub" and `gh issue list` are updated: the round routine, and the open-work list.

### 2.3 CI (after the move)

One workflow on push and on pull request: `pnpm install --frozen-lockfile`, then `pnpm check`, on the Node floor. Where it runs is question 6. No publishing from CI (1.6).

## 3. Units

The round is mostly sequential and small. The reformat touches every file, so it cannot run beside anything else.

| Unit | What | Waits for |
|---|---|---|
| A | Tests type-checked in lib and postcss, the 8 errors fixed | — |
| B | The floor: `engines`, `packageManager`, `.node-version`; `pnpm check` and the leak-check script | — |
| C | Linter and formatter: config, findings measured and ruled, then the reformat commit and `.git-blame-ignore-revs` | A, B |
| D | The purity check, if wanted | C |
| G | Speckit: constitution, house preset, skills committed (section 6) | — |
| E | The move: history across, URL commit, bumps, GitHub archived or mirrored, memories updated | merge of A–D and G; the organisation created |
| F | CI workflow and its runner | E |

A, B and G run in parallel in worktrees. C, D, E and F run one after another. G lands before E, so Codeberg's first commit already holds speckit.

## 4. Risks

- **Lint findings could run to hundreds**, mostly in the plugin. The cap is the rule in 2.1.3: rule off with a reason, no behaviour change. Real defects a rule finds get listed for a later round, not fixed here.
- **The reformat buries `git blame`.** `.git-blame-ignore-revs` fixes that locally and on GitHub. Whether Codeberg's blame view honours it is not checked.
- **Going public cannot be undone.** Forks and caches outlive a later switch to private. The sweep in 1.8 happens before the first push, not after.
- **A runner of one's own runs the workflow's code on that machine.** With a public repository, workflows from forks' pull requests must not run on it without approval.
- **The downstream prototype breaks** on its next install from the GitHub URL, and its issue intake stops, until it switches. That is on the consumer's side of the loop. Toghen only says where it moved.

## 5. Questions

1. **Linter and formatter.**
   a. Biome: one tool and one dependency for both, fast, little config. *Recommended.*
   b. ESLint with typescript-eslint, plus Prettier: the canon, with type-aware rules (floating promises, unsafe `any`), at the cost of several dependencies and slower runs.
   c. A formatter only, no linter.
2. **The purity check.**
   a. A check that reads the forbidden names from a file outside the repository (or a CI secret) and fails on a match. The repository holds the check, never the names. *Recommended.*
   b. A tracked list of names: simpler, but it names the consumer in the repository.
   c. No check; it stays a person's job.
3. **Public or private on Codeberg.**
   a. Public, after the sweep in 1.8: `CLAUDE.md` says "a sibling consumer" instead of the name, and the Figma keys are replaced by the files' names, in history too, through one rewrite before the first push. *Recommended.*
   b. Public, as it is, with nothing rewritten.
   c. Private on Codeberg.
4. **How the history moves.**
   a. Codeberg's migration from GitHub, which also brings the closed issues, the pull requests and their comments. *Recommended* if 3 is not a history rewrite. With a rewrite, the PRs would point at commits that no longer exist, so b.
   b. A plain push of the git history.
5. **GitHub afterwards.**
   a. Archived, read-only, with a README pointing to Codeberg. *Recommended.*
   b. Kept as a mirror, updated by Codeberg on every push.
   c. Deleted.
6. **Where CI runs.**
   a. Woodpecker on `ci.codeberg.org`: hosted, and Codeberg's own pointer for hosted CI. Needs an application and approval. *Recommended.*
   b. A Forgejo runner of one's own on this Mac, as a background service: runs only while the Mac is on.
   c. No CI: `pnpm check` by hand, plus a git hook before each push.
7. **Names.**
   a. Organisation `toghen`, repository `toghen`. A later split reads `codeberg.org/toghen/lib` and so on, and matches the npm scope. *Recommended.*
   b. Organisation `toghen`, repository `Toghen`, as on GitHub today.
8. **The Node floor.**
   a. Node 22 and later, with CI running on 22, so the claim is tested. *Recommended.*
   b. Node 24 and later, the current long-term release.

## Answers (25 October 2026)

1. **a**, Biome.
2. **a**, names read from outside the repository.
3. **a**, public, with one change to the design: the history is not rewritten. It is dropped. Codeberg starts from a single first commit of the swept tree (the consumer's name out of `CLAUDE.md`, the Figma keys out of `.reference`). Versions start again too: "only user consequences, very manageable".
4. **b**, zero history: one commit pushed.
5. **b**, GitHub becomes a mirror, but only once the work is done, and with one GitHub repository per Codeberg repository.
6. "Codeberg": read as **a**, Woodpecker on `ci.codeberg.org`. To confirm.
7. The organisation exists: `Toghen` on Codeberg (public, created 20 October 2026, no repositories yet). The repository's name is still open.
8. "Latest stable Node for the rest": the newest long-term release, today Node 24 (24.22.1, released 20 October 2026). The newest release overall is 26.12.1, not yet long-term. So `engines` says Node 24 and later, and CI runs on 24.

### What the answers change

- **The move becomes a fresh start.** Unit E no longer migrates anything. It builds one orphan commit from the swept tree on `main` and pushes it to an empty `Toghen/<name>` repository, and the local work continues from a new clone. The 625 commits survive only in the old GitHub repository and this checkout. `decisions.md` and the plans remain the record of why things are as they are.
- **Versions start again.** npm never lets a version number be reused. These numbers were never published: lib 0.1.0 and 1.0.0; postcss 0.1.0; query 0.1.0 (published so far: lib 4.1.1 and 6.0.0, postcss 0.18.0 and 0.21.0, query 0.2.0, plus a `0.0.0-stage` each). Starting all five packages at 0.1.0 works. Two steps go with it:
  - `latest` is pointed at the new version explicitly (`npm dist-tag`), since a lower number does not take it on its own reliably.
  - The old versions are deprecated with a message naming Codeberg.

  Anyone pinned to `^6.0.0` stays on 6.0.0 until they change the range.
- **The mirror waits.** No GitHub step in this round beyond leaving the old repository where it is.

## 6. Speckit (added 25 October 2026)

Ruled by Egildo: the constitution comes from `CLAUDE.md`, a preset rewrites the templates in the house register, each round is a speckit feature, and a feature is folded into the living docs at merge.

### 6.1 What speckit 1.2.3 does here

Checked by running `specify init --here --integration claude --script sh` in a scratch folder.

- **It writes two trees.**
  - `.specify/` holds `memory/constitution.md`, the five templates (spec, plan, tasks, checklist, constitution), shell scripts and a workflow file.
  - `.claude/skills/speckit-*/` holds ten skills: constitution, specify, clarify, plan, checklist, tasks, analyze, implement, converge, and one that turns tasks into GitHub issues.
- **`.claude/` is ignored in this repository** (`.gitignore`, for local notes, settings and worktrees). So the skills would never reach Codeberg. The ignore narrows to keep `.claude/skills/` tracked.
- **Features live in `specs/<number>-<name>/`** on a numbered branch, made by `create-new-feature.sh`.
- **A preset** is a folder with a `preset.yml` (`schema_version`, `preset`, `requires`, `provides`). It can replace templates, commands and scripts. `specify preset add --dev <folder>` installs one from a local folder, so the house preset lives in this repository.
- **The stock templates number everything**: requirements, success criteria, user-story priorities, task ids. The stock analyze and tasks commands use those numbers to map requirements to tasks (question 9).

### 6.2 Design

1. **The constitution.** `.specify/memory/constitution.md` takes the rules from `CLAUDE.md`, each said once there:
   - the purity rule;
   - the architectural invariants;
   - the testing lesson (structured shapes, the leak check);
   - the binding status of the rulebook and its four parts;
   - the docs' ownership (reference owns the types; decisions are added to, never rewritten).

   `CLAUDE.md` keeps what is not a rule (what Toghen is, the docs table, the commands, the future developments) and links to the constitution. The constitution names no consumer: "a sibling consumer" stands where the name stood, which is also the sweep question 3 asked for.
2. **The house preset.** Its source is `.reference/speckit-preset/`. Installing it with `--dev` copies it into `.specify/presets/`, which speckit owns, so the source stays outside that folder.
   - *Spec template:* user stories in plain names, acceptance scenarios, edge cases, success criteria, and Toghen's own sections: findings checked against the code with `file:line`, and questions each with context, alternatives and a recommendation, answered in an Answers section.
   - *Plan template:* the constitution check as a gate, then design, units (with what each waits for), risks.
   - *Tasks template:* its last phase is always **Fold**: the outcome goes into `specs.md`, `reference.md` and `guide.md` as each owns it, a dated entry goes into `decisions.md`, and the feature's spec is marked *Folded*. A feature is not done until it is folded. This is what keeps Toghen spec-anchored rather than spec-first.
   - *Analyze command:* also reads the living docs of the packages the feature touches, and reports where they disagree with each other or with the code. The 23 October round's twelve disagreements and the stale "System pane" are the defects it is for.
   - *Feature names:* `specs/<date>-<name>/`, after the plans' own habit, not a running number.
3. **The tasks-to-issues skill** targets GitHub. It is removed, since issues live on Codeberg.
4. **`.reference/plans/`** stays as the archive of the rounds before speckit. New rounds start with `/speckit-specify`.
5. **The round routine** (memory) changes its words, not its shape. The orchestrator writes the spec and plan instead of a plan file, and clarify carries the questions to Egildo one by one. Implement runs the units, and Fold closes the round before the merge.

### 6.3 Unit G

Run `specify init --here --integration claude --script sh`, narrow `.gitignore`, write the constitution from `CLAUDE.md` and trim `CLAUDE.md`, write the preset and install it, remove the tasks-to-issues skill, and update the round routine. Proof: `/speckit-specify` on a throwaway feature produces a spec in the house register, its tasks end in Fold, and the throwaway folder is deleted before the merge.

### 6.4 Risks

- **Stripping the numbers can break the stock commands' bookkeeping** (question 9).
- **A speckit upgrade can change the templates and commands the preset overrides.** `requires` pins the speckit version the preset was written for, and an upgrade is checked like a FigCraft release.
- **Two homes for the rules, during the change.** The constitution and `CLAUDE.md` must never both state a rule. Unit G's review checks it line by line.

### 6.5 Question

9. **The numbers inside speckit's files.**
   a. Keep them inside the files, where the stock commands use them, and never in what is reported to Egildo. The register already allows codes in plans. *Recommended:* the stock analyze and tasks work unchanged, and the preset stays small.
   b. Strip them everywhere and override the commands that map by number. Pure register, but the preset then owns most of speckit, and each upgrade costs more.

**Answer (25 October 2026): 9a.** The numbers stay inside speckit's working papers (`specs/<feature>/`) for the commands' bookkeeping. They never appear in what is reported to Egildo, and never in the living docs the fold writes. The preset therefore replaces the templates and the analyze command only.

## 7. Orchestration (added 25 October 2026)

Egildo asked for a pyramid of agents, with the cheaper models doing the routine work. The rule that sets the pyramid's shape: **a command is cheaper than any agent.** Most of the "copying" in this round is not agent work at all:
- the reformat is `biome format --write`;
- speckit's scaffolding is `specify init`;
- the URL rewrite is a `sed` over the files listed in 1.5;
- the move is one orphan commit and a push.

Those are run by the orchestrator, directly. Agents get only what needs reading or judgment.

### 7.1 The tiers

| Tier | Model | Gets | Why this tier |
|---|---|---|---|
| Orchestrator | Opus (this session) | Writes each agent's brief with exact files and done-conditions, runs the commands, rules on findings, verifies every wave, talks to Egildo | The only place that holds the whole plan |
| Wording | Opus | The constitution and the trimmed `CLAUDE.md`; the review after each wave | Rules are said once, in the house register; a review must not be anchored on the implementer's reading |
| Bounded code | Sonnet | A change with tests, inside one unit's files | Reads code, fixes types, writes a small script or template |
| Fully specified edits | Haiku | Edits whose every line the brief already states | Nothing left to decide |

### 7.2 The waves

Each wave is one workflow run. Agents that change files run in worktrees cut from the integration branch `tooling-codeberg`, and the orchestrator merges each one into it. Before wave 1, the orchestrator creates the branch and commits `specify init`'s output to it, so both speckit agents start from the same scaffolding.

**Wave 1**, four implementers in parallel, then one review:

| Agent | Model | Unit | Files it owns |
|---|---|---|---|
| Tests type-checked | Sonnet | A | `toghen-lib` and `toghen-postcss`: test configs, `typecheck` scripts, the 8 errors |
| The floor | Haiku | B | every `package.json` (`engines`, `packageManager`, the root `check` script), `.node-version`, the leak-check script, whose exact behaviour the brief states |
| Constitution | Opus | G, half 1 | `.specify/memory/constitution.md`, `CLAUDE.md` |
| House preset | Sonnet | G, half 2 | `.reference/speckit-preset/`, `.specify/presets/`, `.gitignore`, the removal of the tasks-to-issues skill |
| Review of wave 1 | Opus | — | Reads the merged diff against this plan and the constitution. Reports; changes nothing |

No two agents touch the same file. B's root `package.json` edit and A's package `typecheck` scripts are separate files.

Gate after wave 1: the orchestrator runs `pnpm check` and the throwaway `/speckit-specify` from 6.3, and fixes or sends back what the review found.

**Wave 2**, sequential, because the reformat touches every file:

| Step | Who | Unit |
|---|---|---|
| Biome installed, config written, findings counted per rule | orchestrator (commands) | C |
| Each rule with findings ruled: safe autofix, or off with a one-line reason; no behaviour change | Sonnet | C |
| `biome format --write`, its commit, `.git-blame-ignore-revs` | orchestrator (commands) | C |
| The purity check: reads names from a file outside the repository, fails on a match | Sonnet, low effort | D |
| Review of wave 2 | Opus | — |

Gate: `pnpm check` green, the plugin rebuilt, no `[object Object]` or unresolved `toghen(` in it. Then the branch waits for Egildo's "merge".

**Wave 3**, after the merge and the three open points (CI, the repository's name, the old GitHub repository):

| Step | Who | Unit |
|---|---|---|
| URL rewrite (`sed` over the list in 1.5), checked by grep | orchestrator | E |
| Versions to 0.1.0 in every package | orchestrator | E |
| Orphan commit of the swept tree, push to `Toghen/<name>`, new clone | orchestrator, on Egildo's go | E |
| Publish at 0.1.0, `latest` moved, old versions deprecated | orchestrator, on Egildo's go | E |
| The Woodpecker workflow running `pnpm check` on Node 24 | Sonnet, low effort | F |
| Memories updated (round routine, open work) | orchestrator | E |

### 7.3 Size

Eight agents in waves 1 and 2, one in wave 3: nine in all, and no single run above five. Three are Opus (the constitution and two reviews), five are Sonnet, one is Haiku. The share of Haiku is small on purpose. Most of what Haiku could do is better done by a command, and the rest needs reading.

## 8. The split shape (added 25 October 2026; supersedes 2.2, 3, 6.3, 7.2 and 7.3)

Ruled by Egildo on 25 October: **one repository per package**, local checkouts in `/Volumes/dev/Codeberg/Toghen` (exists, empty), under the existing Codeberg organisation `Toghen`. Also ruled:
- a `house` repository for what all of them share;
- a local workspace folder linking the checkouts;
- `@toghen/ui` published on npm.

### 8.1 The seven repositories

| Repository | From | Also takes |
|---|---|---|
| `Toghen/lib` | `toghen-lib/` | `.reference/w3c-token-format/`, `.reference/plans/toghen-lib/` |
| `Toghen/postcss` | `toghen-postcss/` | `.reference/plans/toghen-postcss/` |
| `Toghen/query` | `toghen-query/` | — |
| `Toghen/ui` | `toghen-ui/` | — |
| `Toghen/figma` | `toghen-figma/` | `.reference/figma/`, `.reference/figcraft/`, `.reference/plans/toghen-figma/` |
| `Toghen/skills` | `toghen-skills/`, `.claude-plugin/marketplace.json` (its source becomes `./plugins/toghen`) | `.reference/skills-research/`, `.reference/plans/toghen-skills/` |
| `Toghen/house` | new | the shared constitution, the speckit preset, the Biome config, the purity check, the CI file, the local workspace files, `.reference/plans/workspace/`. Each repository copies these, with a line naming the house version they came from |

Names in lowercase, as question 7a recommended. `LICENSE` goes into every repository.

### 8.2 What crosses package folders today

- **The running example.** `toghen-lib/example/` is read by path from other packages:
  - `toghen-postcss`: `test/example.test.ts`, `test/precedence.test.ts`;
  - `toghen-figma`: `test/example.test.ts`;
  - `toghen-query`: `test/ways-in.test.ts`, `sessions/overlay-and-map.session`, and `src/session.ts:26`, which is runtime code.

  After the split, lib ships the example: `example/` goes into `files`, plus an `"./example/*"` export. Today lib exports only `"."`. The readers resolve it through `@toghen/lib`. The example stays lib's, as `model.md` already says.
- **About 110 relative links between packages' docs** (`../toghen-lib/model.md` and the like). They become Codeberg URLs (`https://codeberg.org/Toghen/lib/src/branch/main/model.md`), with anchors kept. A `sed` with one mapping per package does it, and a grep checks that none is left.
- **Dependencies stay as declared**, with `workspace:*` replaced by `^0.1.0`. postcss's peer `@toghen/lib` (`^6.0.0` today) becomes `^0.1.0`.
- **Lockfiles:** each repository gets its own, made with `pnpm install --ignore-workspace`, so CI installs it alone.

### 8.3 The local workspace

`/Volumes/dev/Codeberg/Toghen/pnpm-workspace.yaml` lists the six package checkouts with `link-workspace-packages: true`, so a change across lib and figma stays one step locally. The folder is not a repository: its files are copied from `house` by a command. The cost of the split shows up only at release: publish the producer, then bump its consumers.

### 8.4 The waves

Zero history also removes the reformat's blame problem: the first commit is already formatted. So `.git-blame-ignore-revs` is dropped.

**Wave 1: the monorepo, for what crosses packages.** Runs on the integration branch `tooling-codeberg`, plus the new `house` folder.

| Agent | Model | Owns |
|---|---|---|
| Tests stand alone | Sonnet | lib's and postcss's tests type-checked (the 8 errors), and the example read through `@toghen/lib` in postcss, figma and query (lib's `files` and `exports` included). One agent, because both jobs touch `toghen-postcss/test/example.test.ts` |
| Constitution | Opus | `house/constitution.md`: the shared rules from `CLAUDE.md`, said once, naming no consumer. One short section per package for its own rules (the rulebook for figma; "computes no semantics" for query; reference owns the types, where there is one). Plus the slimmed `CLAUDE.md` each repository will carry |
| House preset | Sonnet | `house/speckit-preset/`, written against the templates `specify init` puts in `house` (run by the orchestrator first) |
| Review | Opus | reads both diffs against this plan; changes nothing |

Commands by the orchestrator: `git init` of `house`; `specify init` there; the docs-link `sed` in the monorepo, after the agents merge; `pnpm -r test` and the plugin's leak check as the gate.

**Wave 2: the split.** Commands only, no agents:
1. Copy each package folder, and its `.reference` parts, into `/Volumes/dev/Codeberg/Toghen/<name>`, then `git init` each.
2. Fix the marketplace path, write the workspace files, `pnpm install` at the parent.
3. Every package's tests run green there.

**Wave 3: the same recipe in six repositories, six agents in parallel.** No worktrees are needed: separate repositories cannot collide. The recipe comes from `house`:
- the Biome config, with each rule that finds something ruled (safe autofix, or off with a one-line reason; no behaviour change), then the format;
- `engines` at Node 24, `packageManager`, `.node-version`, and a `check` script;
- the purity check copied in;
- `specify init`, the house preset, the constitution with the package's section, and the repository's `CLAUDE.md`, with the tasks-to-issues skill removed and `.gitignore` keeping `.claude/skills/`.

| Repository | Model | Why |
|---|---|---|
| figma | Sonnet | 53,000 lines; most lint findings will be here; it also gets the leak check |
| lib, postcss | Sonnet | Real code; findings need reading |
| query, ui | Haiku | Small; the recipe is fully specified |
| skills | Haiku | No code: speckit, constitution, `CLAUDE.md` only |

Then one Opus review across the six. Gate: `pnpm check` in every repository, in workspace mode. Then Egildo's "merge", meaning the first commits are approved.

**Wave 4: release, on Egildo's go at each outward step.**
1. Versions to 0.1.0 everywhere.
2. The seven repositories created on Codeberg (Egildo, or the orchestrator with a token he provides), and each first commit pushed.
3. Publish in dependency order: lib, postcss, ui, query. Move `latest`, and deprecate the old versions.
4. Each repository installed and checked **alone**, from npm, in a fresh clone. This is the proof that the split stands.
5. The CI file: one Sonnet agent writes it in `house`, and the orchestrator copies it out.
6. The old GitHub repository archived; the memories updated.

### 8.5 Size

Twelve agents in all: four in wave 1, seven in wave 3, one in wave 4. Three are Opus (the constitution and two reviews), six are Sonnet, three are Haiku. No single run above seven. Waves 2 and 4 are almost all commands.

### 8.6 Still open

- CI: Woodpecker on `ci.codeberg.org`, as read from "codeberg" (question 6).
- The old GitHub repository: kept private as `toghen-archive` (recommended), since it holds the only copy of the 625 commits outside this Mac.
- Creating seven repositories on Codeberg: by hand, or through its API with a token Egildo provides.

**Answers (25 October 2026).**
- **CI: Woodpecker** on `ci.codeberg.org`, confirmed. Access to it is applied for by Egildo before wave 4, step 5.
- **The seven repositories are created by the orchestrator** with Forgejo's command line, `fj` 0.7.0, already signed in as `egildo` on `codeberg.org` (`fj -H codeberg.org repo create`). Whether it creates into the `Toghen` organisation directly, or needs the API, is checked at wave 4.
- **Still open:** the old GitHub repository (recommended: kept private as `toghen-archive`).

## What wave 1 found (25 October 2026)

Built by three agents, reviewed by a fourth, checked by the orchestrator: `pnpm typecheck` 0; `pnpm test` 0 failures (lib 626, postcss 365, query 30, ui 45, figma 1524); the plugin builds, with no `[object Object]` and no `toghen(` outside one doc comment.

- **Tests type-checked.** The 8 errors were fixed without changing an assertion and without a cast to `any`. The two wrong-arity calls were tokens meant to be untyped; they now pass `''`, as the resolver itself writes an untyped token (`toghen-lib/src/resolver/resolver.ts:1024`).
- **The example through the package.** lib ships `example/` and exports `./example/*`. A session's path starting with `@` now names a file inside a package (`session.ts`, `sourceAfter`). That is a change to the session format: query's `specs.md` and `guide.md` now say so, and `decisions.md` has a dated entry. Paths written from the monorepo root in lib's and query's guides now read `node_modules/@toghen/lib/example/...`.
- **The constitution** is in house at `.specify/memory/constitution.md`, the only copy there. The review found a second copy at house's root and removed it. Every rule of `CLAUDE.md` is stated once, and the consumer is named nowhere. The sentence about the local workspace moved out to house's `CLAUDE.md`: it describes an arrangement, not a rule. The rulebook rules stay in the shared section: each repository carries the whole text anyway.
- **The preset** is installed and resolves on 1.2.3. The review's fixes are applied:
  - the plan template gained a *Technical context* section, which the stock plan and tasks commands read;
  - the analyze layer runs before its report, inside it;
  - the Fold check asks for `reference.md` only where a package has one.

  Feature folders are named by date through a layer on the specify command, since 1.2.3 numbers folders inside that command, not in the script.
- **The tasks-to-issues skill** was deleted by hand, with its manifest entry. Speckit offers no option to skip it. A later `specify integration upgrade` brings it back.

Ruled by the orchestrator, as consequences of answers already given:
- **The constitution skill is removed in every package repository.** It edits the constitution in place, and the governance says it is amended only in house. It stays in house.
- **The docs-link rewrite moves to wave 2**, done per repository after the copy. Links inside one package that point into `.reference/` can only be made right once `.reference` has moved with it. The rewrite maps every monorepo path to its repository and path: relative when it stays in the same repository, a Codeberg URL when it crosses. `README.md` links stay absolute, since npm renders them.

Left for later, by name:
- query resolves the example when it starts, so it needs a lib that exports `./example/*`. The release order (lib first) covers it.
- The stock tasks command still suggests a final "Polish" phase. The template and the analyze check both insist on Fold. If a real round ends on Polish, a small layer on the tasks command follows.

**The throwaway feature (25 October 2026).** An agent followed `/speckit-specify` in house, for a made-up `version` command for query. It produced `specs/2026-10-25-version-command/spec.md` from the house template: Findings, stories, requirements and success criteria with their numbers, one Question with three alternatives and a recommendation, and an empty Answers section. The folder was then deleted. Two rough edges, for later:
- the stock checklist the specify command writes still asks for "no implementation details", which sits badly beside Findings with `file:line`. A layer on the checklist follows if it gets in the way;
- the Answers heading's date is filled in only when the owner answers. The template's comment should say so plainly.

## What wave 2 found (25 October 2026)

The split was made from `tooling-codeberg` at "Record the throwaway speckit feature's result". From here on, this copy of the plan, in house, is the live one; the monorepo's copy stops there.

- **Seven folders, each a repository with one local commit**: lib 116 files, postcss 112, query 26, ui 54, figma 323, skills 22, house 57. A script copied every tracked file to its repository, `.reference` parts included. It rewrote the markdown links that cross folders:
  - a link that stays in the same repository stays relative (`../.reference/figma/…` became `.reference/figma/…`);
  - a link that crosses repositories became a Codeberg URL;
  - links to the old GitHub repository became Codeberg URLs.

  Not rewritten: links inside the vendored W3C reports that point at sibling reports never copied (already broken in the monorepo), and paths written in backticks in the archived plans.
- **Manifests**: every package at 0.1.0, every `@toghen/*` range at `^0.1.0` (postcss's peer included), and the repository, homepage and bugs fields on Codeberg. `@toghen/ui` is no longer private.
  - The dev tools that came from the monorepo's root are now declared where used: `@types/node` everywhere, `tsx` in lib and postcss, `typescript` in ui and figma.
  - query's purity test lists its dev dependencies exactly, so it now lists `@types/node` too.
- **The marketplace** source is `./plugins/toghen`, and both `claude plugin validate` runs pass.
- **lib's guide** installs from `git+https://codeberg.org/Toghen/lib.git#main`. The sentences saying npm cannot install from a folder, and that the repository is private, are gone.
- **`.gitignore`**: the monorepo's in each repository. lib's 90-line one, inherited from a template (Angular, Nx, Storybook…), is replaced, keeping only `.typedoc/`.
- **The local workspace** files are `house/workspace/`, copied to `/Volumes/dev/Codeberg/Toghen`. They link the five packages, and allow esbuild's install script, which pnpm 10 blocks otherwise.
- **The gate, in the workspace**: `pnpm build`, `pnpm typecheck` and `pnpm test` all pass. The tests number as many as in the monorepo: lib 626, postcss 365, query 30, ui 45, figma 1524. The plugin has no `[object Object]`, and `toghen(` appears only in one doc comment.

Waits for the publish in wave 4: each repository's own lockfile (`pnpm install --ignore-workspace`), since `@toghen/lib@0.1.0` is not on npm yet.

## What wave 3 found (25 October 2026)

**By command, in every repository:**
- Biome 2.5.15, pinned to a release older than two weeks. The shared config is `house/shared/biome.jsonc`:
  - two-space indent (275 files already used it, against 96 at four) and single quotes (all files did);
  - 140 columns;
  - CSS not even parsed, since the `toghen()` directives are not CSS until postcss resolves them, and JSON left alone.
- Two recommended rules are off by count: the non-null assertion (about 1,100 places) and explicit `any` (about 300).
- All code formatted.
- `engines` at Node 24, `packageManager` pnpm 10.29.3, `.node-version`, and `pnpm check`: lint, typecheck, test, build, purity, plus figma's leak check.
- The purity check in `scripts/`. Its names live in `~/.config/toghen/purity-names`. It found three Figma file keys in figma's `.reference`, one of them not known before, and they are gone. All seven repositories pass it.
- Speckit with the house preset. The constitution is stamped "Copied from house, constitution version 1.0.0", and each repository has its own `CLAUDE.md`. The tasks-to-issues and constitution skills are removed, with their manifest entries. `.gitignore` keeps `.claude/skills/` tracked.
- query's dependency test now lists `@biomejs/biome` as well.

**By agents:** lib, postcss and figma on Sonnet; query and ui on Haiku. They ruled each lint rule that still had findings. Every edit that remains is behaviour-neutral: an Opus review read every hunk.

Fixed after the review:
- figma's leak check passed on an empty `dist/`. It now needs `code.js`, `ui.html` and at least one CSS or HTML file.
- Two `[^]` in ui's tests had been filed as defects. They now read `[\s\S]`, which matches the same.
- An optional chain in lib's tests is now `!`, which throws the same TypeError.
- figma's `noPrototypeBuiltins` override names its six files instead of two folders.

**The one real defect found, for a later round:** lib's `test/unit/resolver/factoring/factoring.property.test.ts` counts the cells it compares in `checked` but never asserts it. The property test could pass having compared nothing. The fix is `assert.ok(checked > 0)`. It is suppressed at that site and labelled.

**The gate:** `pnpm check` exits 0 in lib (626 tests), postcss (365), query (30), ui (45) and figma (1524, with "leaks: none"), run one at a time. skills passes its purity check and `claude plugin validate`.

**Noted:** Biome 2.5 reports `"recommended"` as deprecated, in favour of a `preset` field, as an info line that does not fail. `biome migrate` rewrites it when house next moves Biome.

## Wave 4, first half: on Codeberg (25 October 2026)

- **Seven public repositories** were created in the `Toghen` organisation with `fj org repo create`: lib, postcss, query, ui, figma, skills, house.
- **Each holds one commit.** The tree is the same as the last local commit, checked by diff. The local history is kept on a branch `local-history` in each checkout, never pushed.
- **Checks before the push:** the purity check passed in all seven, and a scan for npm or GitHub tokens and private keys found nothing.
- **After the push:** each remote `main` matches the local commit, and a cross-repository docs link (`lib/src/branch/main/model.md`) resolves.
- **npm, still to do, on Egildo's go.** The npm account has two-factor authentication for writes, so every publish, tag and deprecation asks for a one-time code. Egildo types it; the orchestrator does not. The steps:
  1. Build, then publish at 0.1.0 in dependency order: lib, postcss, ui, query. figma stays off npm.
  2. Point `latest` at 0.1.0 wherever it still names an old version.
  3. Deprecate every older version with a message naming Codeberg: lib `0.0.0-stage`, 4.1.1 and 6.0.0; postcss `0.0.0-stage`, 0.18.0 and 0.21.0; query `0.0.0-stage` and 0.2.0.
  4. Make each repository's own lockfile against npm (`pnpm install --ignore-workspace` in a fresh clone), check each repository alone, and push.

## Everything at 0.1.0 (25 October 2026)

Ruled by Egildo: "everything 0.1.0, also figma", and a force-push to fold it into each first commit.

- **figma goes on npm**, at 0.1.0. It gained:
  - a `files` list: `cli`, `main`, `shared`, `dist`, `manifest.json`, the README and the user docs, 118 files and 1.3 MB packed;
  - `tsx` as a runtime dependency, since the command runs TypeScript;
  - `prepack` building the plugin;
  - a README.

  Proved from packed tarballs of lib and figma, installed in an empty project: `toghen-figma check` on lib's shipped example and a component file read it with no errors. The guide's "the package is private" note became `pnpm add -D @toghen/figma`.
- **The skills plugin** went from 0.5.0 to 0.1.0. Every pin names the new numbers, with the old ones beside them, and paths name the repository (`figma/guide.md`), not the monorepo folder.
- **House's stamps** went from 1.0.0 to 0.1.0: the constitution, the preset, and every "Copied from house" line.
- **Each package's `decisions.md`** has a dated entry for the move and the renumbering, naming the last version before the split. ui's specs no longer call it private.
- **Gate:** `pnpm check` exits 0 in all five code repositories, with the same test counts. skills passes its purity check and both plugin validations.
- **All seven repositories were force-pushed**, each still one commit.

npm, on Egildo's go: publish lib, postcss, ui, figma, query at 0.1.0, in that order; check `latest`; deprecate the older versions.

## npm (25 October 2026)

- **Published at 0.1.0, `latest` on 0.1.0 everywhere:** lib, postcss, ui, figma, query. npm refuses to give `latest` implicitly to a version below an earlier one, so the publish names `--tag latest`. Egildo approved each publish in the browser.
- **ui and figma show a `0.0.0-stage` beside 0.1.0,** dated seconds before it, as lib, postcss and query already did. It was not published by hand. Where it comes from is not checked.
- **Each repository checked alone**, in a fresh clone from Codeberg, installing its neighbours from npm. `pnpm check` passes in all five: lib 626, postcss 365, query 30, ui 45, figma 1524. The lockfile each install made is folded into its repository's one commit and force-pushed.
- **Found by that check:** figma's style test imported `@toghen/ui/build.mjs`. That script works only inside a checkout of ui: it reads fonts from ui's own `node_modules`, and it imports ui's dev tools. The test now reads `@toghen/ui/styles.css`, the stylesheet ui ships, which is the same text the script builds. **Left for a later round:** ui still exports `./build.mjs`. It should either stop exporting it, or carry what it imports as dependencies.
- **The deprecations:** the first try answered 404. The cause was not the code: the token npm had stored no longer authenticated, and npm reports a refused write as 404. Publishing had worked only because it signs in through the browser every time. After a fresh `npm login`, `npm deprecate` asked for the browser sign-in like the publish did. Done, all with "Toghen moved to Codeberg and restarted at 0.1.0" and the repository's URL: lib 4.1.1 and 6.0.0, postcss 0.18.0 and 0.21.0, query 0.2.0. Every 0.1.0 is clean, and `latest` is 0.1.0 for all five.

## GitHub mirrors and CI, until Woodpecker (25 October 2026)

Ruled by Egildo: CI through GitHub mirrors under his own account, no organisation, and "deal with the old toghen-lib name".

- **The old private `egildo/toghen-lib`**, the library before the monorepo (last pushed June 2026), is now `egildo/toghen-lib-old`. It is still private and otherwise untouched. `egildo/Toghen` stays as it is.
- **Seven public mirrors**, `egildo/toghen-<name>`. Each description says it is a mirror of `codeberg.org/Toghen/<name>`, where issues and changes go, and issues and the wiki are off. GitHub offers no switch for pull requests.
- **How they stay in step:** each checkout's `origin` has two push addresses, Codeberg then GitHub, so every `git push` from this Mac reaches both. No token went into Codeberg. The limit: a push from anywhere else reaches Codeberg only.
- **The CI:** `house/shared/github/check.yml`, copied to `.github/workflows/check.yml`. It runs `pnpm install --frozen-lockfile` and `pnpm check` on the Node version in `.node-version`. skills and house run only the purity check, from their own variants.
- **First runs:** green in all seven. figma ran 1524 tests and found no leaks.
- **`PURITY_NAMES`**, on Egildo's word, is a secret on each of the seven mirrors, set with `gh secret set` straight from `~/.config/toghen/purity-names`. Rerun with it, every mirror's purity check reads the names and passes. Change the list in that file, then set the secret again on all seven.
- **Publishing from the mirrors** (ruled by Egildo, 25 October 2026): `house/shared/github/publish.yml`, copied to `.github/workflows/publish.yml` in the five code repositories. A pushed tag `v<version>` runs it: it refuses a tag that is not the version in `package.json`, runs `pnpm check`, then `npm publish`. npm trusts the workflow by name (trusted publishing), so no token is stored and nobody signs in; each version records the commit it was built from. npm offers this to GitHub Actions, GitLab and CircleCI only, not to Codeberg, which is why it runs on the mirror. Each package names its trusted publisher once on npmjs.com (owner `egildo`, repository `toghen-<name>`, workflow `publish.yml`). A release: bump the version, commit, `git tag v<version>`, `git push origin main v<version>`.
