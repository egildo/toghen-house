# Plan: the five leftovers of the measurement of 26 November 2026

**Status**: Ruled. Egildo answered the two questions on 26 November 2026, "1a 2a"; see *Answers*.

## Where this comes from

On 26 November 2026 the `toghen` skills were measured again, against figma 0.1.18, lib 0.2.1 and query 0.1.2 from npm, with FigCraft headless through `figcraft-cli` 0.18.1. The record is `skills/decisions.md`, *Measured again: figma 0.1.18, lib 0.2.1, query 0.1.2 (26 November 2026)*, under *What the measurement found in the tools*. It lists five behaviours: four in figma's command line, one in query's. Egildo said "Fix them" the same day.

Every `main` is at its released tag (figma `v0.1.18`, query `v0.1.2`), so the npm packages are the code on `main`.

The measurement's logs are under `/private/tmp/claude-501/-Users-egildo--claude-spares-spare-Fpfl06/85778d0b-8740-48e9-8b29-268e30e7de4d/scratchpad/measure-2026-11-26/logs/`, beside the copy of the worked example it ran (`mini/`). They may be gone by the time you read this. If so, read the code and write your own probe.

**Obvious** means what it meant on 23 November: the defect is proved, and the fix follows from a rule Toghen already has. Anything else is a question below.

## The findings

1. **figma's line for a missing entry names the path twice.** Measured: `tokens/nope.resolver.json: the entry document "/…/tokens/nope.resolver.json" could not be read. ENOENT: no such file or directory`. The prefix is figma's: `run.ts:313`, `check.ts:31` and `members.ts:30` print `${entryLabel(entry)}: ${message}` for anything `readSource` throws. The quoted location is lib's, from `readJson` (`lib/src/resolver/load.ts:374`), and it is absolute because `readSource` makes the entry absolute before `load` (`run.ts:109`). Each copy follows ruling 2a of 25 November: the prefix is as typed, and lib's location is "a location lib read". query prints lib's line alone, after `error: `, so its path shows once. The prefix still matters for an error that names no file, such as `$ref "…" could not be resolved within the document` (`lib/src/resolver/resolver.ts:846`). Needs a ruling: question 1.
2. **`lock --dry-run` says a run would publish the library.** Measured: `Dry run: "app" has 0 updates from design/kit.figcraft waiting; a run would publish the library first, then accept them.` `lock` publishes nothing: in `finishFigcraftLibrary` (`run.ts:466`), `library` is undefined for `lock`, and only a run that builds publishes. `describeTarget` (`cli/figcraftLibrary.ts:156`) prints the same tail for both. Obvious: a dry run says what the real run would do, as its other lines do (`reference.md:1614`).
3. **query prints its usage after a document that does not load.** Measured with `--doc tokens/nope.resolver.json check`: lib's line, then the usage, exit 2. The arguments were right. The outer handler (`src/main.ts:96-101`) appends the usage to every error it catches. Usage errors never reach it: `argsOf` failures are caught in `main` (`:51`) and print the usage there, exit 1. What reaches the outer handler is a load that threw (exit 2, `specs.md:29`) and the missing source, which already names `--doc` (`test/ways-in.test.ts:104`). Obvious: the specs give the usage to usage errors only.
4. **An unchanged run counts one step.** Measured: `Chip: 1 step · 0 created · 0 updated · 0 deleted`, while the unchanged artboard prints `Chip reference: 0 steps · …`. The step is `RELAYOUT`: the set's diff ends with it always (`shared/components/diff.ts:596`), and the real run re-lays the members out (`main/components/update.ts:264`), which moves nothing when nothing changed. The line counts `report.steps.length` (`cli/run.ts:706`). Only the command line counts steps. Needs a ruling: question 2.
5. **A first headless library run names its target "Untitled".** Measured: `Dry run: design/kit.figcraft is not enabled in "Untitled"`, `Enabled design/kit.figcraft in "Untitled"`, `The lock in "Untitled" holds "kit" …`. Every later run says `"app"`. A headless session on a file that does not exist writes into its untitled file (`cli/figcraft.ts:258`), and the save gives it the file's name, after these lines. A new library file is already named after its path (`pinPath`, `cli/figcraft.ts:380-383`, `file.new {name}`). Obvious: the target gets the name the save will give it, before the run says anything of it. FigCraft 0.18.1 has `file.rename {name}`.

## Questions

1. **How should figma print a missing entry?** (finding 1)
   - *Context*: lib's message names the file it could not read, as it read it, which is absolute. figma adds the path as typed in front. The prefix is needed for lib's errors that name no file.
   - a. **figma prints lib's line alone when it already names the entry's location**, as query does: `the entry document "/…/tokens/nope.resolver.json" could not be read. ENOENT: …`. Every other load error keeps the prefix. Costs: the path shows absolute, though it was typed relative. *Recommended*: one copy, the same line in both tools, and ruling 2a already lets a location lib read stay as it is.
   - b. **figma and query read the entry themselves first** and print it as typed, as figma does for a component file: `tokens/nope.resolver.json cannot be read: ENOENT: …`. Costs: a second form for one failure; an entry that is not JSON is still lib's line, so figma still doubles that one.
   - c. **Leave the code, and have the reference show the line as it is.** Costs: the double stays.

2. **What should a set's line count when nothing changed?** (finding 4)
   - *Context*: every set's plan ends with `RELAYOUT`, so a set never counts fewer than one step; an artboard's plan has none. The count is printed only by the command line, and the docs' examples include it (`reference.md:1537`, `guide.md:798`).
   - a. **The count leaves out `RELAYOUT`.** An unchanged set prints `Chip: 0 steps · 0 created · 0 updated · 0 deleted`, as the artboard does. The run still re-lays the set out. Costs: every count drops by one, so the docs' examples and the tests that pin a count change. *Recommended*: the line counts what the run changes, and the set and the artboard read alike.
   - b. **The line drops the step count,** and keeps created, updated, deleted, breaking and drifted. Costs: a run of only patches to properties, which no member count shows, reads `0 created · 0 updated · 0 deleted`.
   - c. **The diff plans `RELAYOUT` only when another step changes the set.** Costs: a member a person dragged out of place is no longer put back by an unchanged run; a behaviour change, for a line.

## Answers

Egildo, 26 November 2026: "1a 2a".

1. **a.** figma prints lib's line alone when it already names the entry's location, as query does; every other load error keeps the prefix.
2. **a.** A set's count leaves out `RELAYOUT`; the run still re-lays the set out.

## Rules that bind every unit

As in `plan-2026-11-25-measurement-fixes.md`, *Rules that bind every unit*: read first, purity, test first in the W3C structured shapes, decisions written by the integration from the reports, the doc that owns the statement, the code's own style, the root fix and the smallest diff, stop on an item that turns out to need a ruling, and commit messages in the house style ending with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.

## The repositories, the branches and the worktrees

As on 25 November: the checkouts in `/Volumes/dev/Codeberg/Toghen`; one worktree per unit under `/Volumes/dev/Codeberg/Toghen-worktrees/`; each branch cut from the repository's `main`; commit on the branch, and never push, tag, publish, merge or touch `main`; `pnpm install --frozen-lockfile` inside the worktree; the gate is the repository's own `pnpm check`, run inside the worktree.

| Unit | Repository | Worktree folder | Branch | Starts |
|---|---|---|---|---|
| F | figma: command line | `figma-cli` | `leftovers` | Items 2 and 5 at once; 1 on question 1; 4 on question 2 |
| Q | query | `query` | `leftovers` | At once |

No unit touches lib: none of the five needs a change there.

## F. figma: the command line

Your files are `cli/`, the command line's tests, `test/support/fakeFigcraft*` and `fakeLibraries.ts`, and the sections of `reference.md`, `specs.md` and `guide.md` on the command line, the lock and FigCraft.

1. **A missing entry, as question 1 rules.** On a: the three callers print the message alone when it already names the entry's location, through one helper beside `entryLabel`. Test a missing entry, an entry that is not JSON, and a `$ref` within the document that resolves to nothing, which keeps the prefix. The reference's rows for the entry follow.
2. **`lock`'s dry run.** `describeTarget` is told whether the run publishes. For `lock`, the tail says what `lock` would do: accept the updates and write the lock. A run that builds keeps today's line. Test both in the fake (`test/figcraftLibraries.test.ts:248` pins today's line). The reference's dry-run lines (`reference.md:1614`) follow.
3. **The step count, as question 2 rules.** On a: `cli/run.ts:706` counts the steps that are not `RELAYOUT`. Change the counts in the tests and in the docs' examples, and only the counts.
4. **The first name of a headless target.** When the file does not exist, the session's untitled file is renamed to the file's name without `.figcraft` before it is pinned, as `pinPath` names a new library file. Measure with `figcraft-cli` 0.18.1 that `file.rename` and the save agree on the name, and that a dry run writes nothing to disk. Test in the fake: the first run's lines name the target as later runs do.

## Q. query

Your files are `src/main.ts`, `test/ways-in.test.ts`, and the exit-code paragraphs of `specs.md` and `guide.md` if they say otherwise.

1. **No usage after a load that throws.** The outer handler prints `error: <message>` and exits 2, without the usage. Extend the test at `test/ways-in.test.ts:201` so the usage is absent, and keep the missing-source test at `:104` as it is.

## Integration

My review of each branch, each repository's `pnpm check` green on the merged result, and one entry per repository in its `decisions.md`, from the reports. Merging waits on Egildo's word. Then figma 0.1.19 and query 0.1.3 are bumped, tagged and pushed, and published by trusted publishing; the skills are re-pinned and measured as `skills/specs.md` says.
