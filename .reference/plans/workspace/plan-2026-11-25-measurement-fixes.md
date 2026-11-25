# Plan: the fixes from the measurement of 25 November 2026

**Status**: Draft. Five questions are open. A unit or item that waits on one starts once Egildo answers it.

## Where this comes from

On 25 November 2026 the `toghen` skills were measured against the released tools: figma 0.1.17, query 0.1.1, postcss 0.2.0 and lib 0.2.0, from npm, with FigCraft headless through `figcraft-cli` 0.18.1. The measurement is in `skills/decisions.md`, *Measured: the skills against the released tools (25 November 2026)*. It found seven behaviours of figma and query. This plan proves each one on `main`, finds its cause, and says what fixes it.

Every `main` is at its released tag (figma `v0.1.17`, query `v0.1.1`, lib `v0.2.0`), so the npm packages are the code on `main`.

The probes are under `/private/tmp/claude-501/-Users-egildo--claude-spares-spare-Fpfl06/85778d0b-8740-48e9-8b29-268e30e7de4d/scratchpad/plan-2026-11-25/`: a copy of the worked example (`mini/`), the same folder moved (`machineB/`), and small synthetic documents (`dangling/`, `samename/`, `relprobe/`). They may be gone by the time you read this. If one is, read the code and write your own probe.

**Obvious** means what it meant on 23 November: the defect is proved, and the fix follows from a rule Toghen already has, whether in the constitution, a package's docs, the W3C specs or the package's own stated intent. Anything else is a question below.

## The findings

1. **The lock holds a FigCraft library's path absolute.** Still true. `run.ts:374` makes the path absolute for FigCraft, and `lockedFrom` (`cli/figcraftLibrary.ts:52-54`) stores that same path. Nothing reads it back: `main/utils/lockStore.ts:25` only checks that it is a string. Measured today:
   - a checkout moved to a new folder, with the old one gone, still resolves through the lock in a dry run, and a real `--use` run points the library at the new path by its id and changes nothing else;
   - FigCraft stores its own link absolute even when it is given a relative path (`library.add {path: "design/kit.figcraft"}` stored `/…/design/kit.figcraft`). So the file holds this machine's folder whatever Toghen writes, and a run on another machine rewrites it.

   Layer: figma's command line. Needs a ruling: question 1.
2. **A missing entry document's message repeats the path.** Still true. The figma command exits 1, as its reference says; `toghen-query` exits 2, as its specs say for a load that throws. The cause is lib's: `load` gives the entry the label `the entry document "<path>"` (`src/resolver/load.ts:118, 126`), and `readJson` adds `: "<location>"` after it (`:376`). For the entry the two are the same path. lib's own test pins the double (`test/unit/resolver/load.test.ts:321`). Obvious: lib's message names the reference, the file it was written in and the location read, so the entry is named once. Node's `open '<path>'` at the end, from the readers in `figma/cli/bin.mjs:16` and `query/src/load.ts:32`, is the third copy. It goes with question 2, since it is how the command line prints a path.
3. **A dry run after a real run says it would restack, and the real run moves nothing.** Still true, headless and in every target. `describeStack` (`shared/components/stack.ts:114-124`) names every set still where Toghen put it, which after any run is all of them. It never asks `restack()` whether any would move. The real run calls `restack()` (`main/components/endOfRun.ts:213`) and moves nothing. The real run is right. figma's `decisions.md` noted it on 23 November, under *Noticed, not fixed*. Layer: figma's shared code. Needs a ruling: question 3.
4. **The library lines mix absolute and relative paths.** Still true. `publish`, `enable` and `describeTarget` (`cli/figcraftLibrary.ts:87, 92, 123, 127, 130, 137-140`) print the path made absolute for FigCraft. `Saved` prints FigCraft's answer (`cli/figcraft.ts:266`). `Building`, `Using`, the lock line and `Writing into` print what was typed (`run.ts:386-387, 414, 484, 368`). The code's own rule elsewhere is "as typed, for messages": `entryLabel` (`run.ts:104`) and a component file's `fileName`. Layer: figma's command line. Needs a ruling: question 2.
5. **`--tokens <file>…` reads as several files after one flag.** Still true. The parser takes one file per flag (`src/main.ts:22`, `multiple: true`), and the first word that is not a flag starts the query (`:36`), so a second file is `unknown command '<file>'`. Obvious: query's guide already says "Repeat the flag for more files" (`guide.md:105`). The usage text (`src/main.ts:17`) and `specs.md:48` disagree with it. Taking several files after one flag is not chosen: a file name and a command word cannot be told apart without guessing.
6. **`toghen-query css surface.page` answers `--page`.** Still true. `css nope.nothing` answers `--nothing` too, exit 0. The `css` form calls `cssPropertyName(path)` (`src/commands.ts:334`). That is a pure function: it drops the first segment, whatever it is (`lib/src/utils/css-name.ts:42-44`), and knows no collection. Every other lookup takes the one-collection shorthand (`DesignSystem.lookupPath`), so `value surface.page` works. query's guide says so (`guide.md:76`). query cannot check the path itself: that is semantics, or a second export. Layer: lib, or query's rule. Needs a ruling: question 4.
7. **`toghen-query check` passes a reference to nothing.** Still true, in the structured shapes too: a colour whose `$value` is `{nope}` and a shadow whose `color` is `{color.missing}` pass `check`, exit 0, while `value` throws `UnresolvableReferenceError` for each. `Collection.check()` (`lib/src/core/collection.ts:926`) looks at modes, chains and names, never at references. lib's specs list it under *Future*: "Dangling references in the document-wide check" (`specs.md:1040`). query's own `a-broken-document` session calls its `check` "Everything wrong with the document, in one call", and leaves out the dangling `card.border`. Layer: lib. Needs a ruling: question 5.

Found while proving them:

8. **A dry run says a moved library is not enabled.** In the moved folder, `kit:check` printed `Dry run: …/kit.figcraft is not enabled in "app"; a run would enable it.` It is enabled, at the old path. `describeTarget` calls `listed` without the id (`cli/figcraftLibrary.ts:136`), so `listed` matches by path (`:105`). Obvious: `listed`'s own comment says "by its id when known", the guide says "A moved library is the same library", and a dry run says what the real run would do.
9. **`toghen-query check` hides two tokens with one custom property.** `space.a.b` and `space.a-b` give `--space-a-b`. lib's `check()` lists them in `sameCssName`. query's printer drops the list and exits 0 (`src/commands.ts:296-321`); only `--json` shows it. Obvious: query prints what the core answers, and its specs fail `check` on "the lists the core calls what is wrong with the collection's tokens". `sameCssName` is one of them.

## Questions

1. **What should the lock hold for a FigCraft library's path?** (finding 1)
   - *Context*: Nothing reads the lock's `path`. Every real run is given the path again by `--library` or `--use`, and `library.add` points the library there by its id. FigCraft's own link in the same file is absolute and stays so whatever Toghen passes it. The ruling of 23 October 2026 (figma `decisions.md`, *Ruled: libraries in FigCraft*, 2) says the entry "holds the path and the id".
   - a. **The path relative to the target file's folder.** Costs: the target's own path is needed, which live FigCraft does not always list; the file still changes on another machine, through FigCraft's link.
   - b. **No path: the entry keeps the id, which is what finds the library.** A `path` in an older file is still read, and ignored. Costs: the October ruling changes; a person reading the lock sees no path, though FigCraft's link still shows one. *Recommended*: nothing needs the path, and the lock stops carrying the home folder of whoever ran last.
   - c. **Keep it absolute, and say so in the reference.** Costs: the lock carries that folder, and changes on each machine.

2. **How should the command line print a path?** (findings 4 and 2)
   - *Context*: The code's rule is "as typed, for messages", and the library lines break it. A read failure also ends with Node's own copy of the path. The skills now teach `Saved <absolute path>.`
   - a. **As typed, wherever a person typed it.** Published, Enabled, the dry-run lines and Saved print what was given. A path nobody typed, such as a folder the run derived or a location lib read, stays as it is. The readers in figma and query give Node's reason without its path (`no such file or directory`). Costs: `Saved` changes, so the skills change back. *Recommended*: it is the rule the code already follows, and lines read the same on every machine.
   - b. **Absolute everywhere,** `Writing into`, `Building`, `Using` and the lock line too, with the readers as in a. Costs: long lines that differ per machine; most message tests change.
   - c. **Leave the code, and have the reference say which lines are absolute.** Costs: one run keeps mixing both.

3. **What should a dry run say of the restack?** (finding 3)
   - *Context*: The docs give no positions in a dry run, because the sets a run builds have no size yet (specs, *Toghen's own sets are restacked*). The real run is right; the dry run's line is noise after every run.
   - a. **Run `restack()` at today's sizes, and name only the sets it would move.** With nothing to move, say nothing. Costs: a set the run makes larger can push others further than the dry run said; the line still gives no positions. *Recommended*: the dry run and the real run then use one function, and agree whenever the sizes do not change.
   - b. **Keep the line, only on a page where the dry run creates, updates or deletes a set or artboard.** Costs: on such a page it still names sets that will not move.
   - c. **Keep it, and reword the line and the docs:** Toghen places these sets, and a run stacks them again. Costs: the line stays on every dry run.

4. **How should `css` take a path without its collection?** (finding 6)
   - *Context*: `cssPropertyName` is a pure function, and stays one: the adapters give it lookup paths they already hold. Lookups in the core take the one-collection shorthand; the custom-property name does not. query's one rule allows one export per form, and the constitution names the three forms that read more than one.
   - a. **lib: a `DesignSystem` method answers a token's custom property by its path,** with the shorthand, and a path that names no token throws. query's `css <path>` calls it. Costs: one lib export, and a lib release before query's. *Recommended*: the core answers, the shorthand holds everywhere, and a typo gets an error.
   - b. **query: `css <path>` calls `lookupPath`, then `cssPropertyName`,** and says so in its last line. Costs: house amends the constitution's query rule for a fourth such form; a path that names no token still gets a name.
   - c. **No code: the usage reads `css <collection.path>`,** and the guide keeps its sentence. Costs: the wrong answer stays silent.

5. **Should `check` report a reference to nothing?** (finding 7)
   - *Context*: The W3C format says a tool must report an unresolvable reference, and must validate reference targets (§7, as lib's specs cite it in *Failures are errors, not warnings*). It also says to resolve only when a value is needed. lib throws when it resolves such a value. Only query calls `check()` today.
   - a. **lib: `check()` gains `unresolvable`:** each token that throws when resolved in a mode, with the mode, the reference and the error's reason. A cycle goes there too, since §7 lists both. query prints the section and exits 1 on it. Costs: `check` resolves every token in every mode, which a large product build makes slow, to be measured; `Check` gains a field. *Recommended*: `check` becomes what it says, and lib's *Future* line is done.
   - b. **lib: `Resolver` refuses such a document when it builds.** Costs: against "resolve only when needed"; one bad token stops every tool, a figma sync of the rest included.
   - c. **No code: query's guide says `check` does not follow references,** and `value` does. Costs: `check` keeps passing a document the standard says must be reported.

## The repositories, the branches and the worktrees

- **The checkouts.** The repositories are the checkouts in `/Volumes/dev/Codeberg/Toghen`.
- **One worktree per unit,** under `/Volumes/dev/Codeberg/Toghen-worktrees/`, outside the checkouts' folder, whose `pnpm-workspace.yaml` would capture a nested worktree.
- **Branches.** Each branch is cut from the repository's `main`.
- **Commits.** Commit on the branch. Never push, tag, publish or merge, and never touch `main`.
- **Installing.** Inside the worktree, `pnpm install --frozen-lockfile`. The packages resolve their neighbours from npm there, as CI does. So query's items that need lib's new exports wait until lib is on npm.
- **The gate.** The repository's own `pnpm check`, run inside the worktree. A unit is done only when its gate is green.

| Unit | Repository | Worktree folder | Branch | Starts |
|---|---|---|---|---|
| A | figma: command line | `figma-cli` | `measurement-fixes-cli` | Item 1 now; 2 on question 1; 3 on question 2 |
| B | figma: the dry run's restack | `figma-restack` | `measurement-fixes-restack` | On question 3 |
| L | lib | `lib` | `measurement-fixes` | Item 1 now; 2 on question 4a; 3 on question 5a |
| Q | query | `query` | `measurement-fixes` | Items 1-2 now; 3-4 once lib's release is on npm |
| H | house | `house` | `measurement-fixes` | Only on question 4b |
| S | skills | `skills` | `measurement-fixes` | After the releases |

## Rules that bind every unit

- **Read first.** Read the constitution (`.specify/memory/constitution.md`), the repository's `CLAUDE.md`, and the docs of the package you change, before you change it.
- **Purity.** No consumer's name, palette, token paths or components, anywhere: source, tests, docs or commit messages. Synthetic tokens only. `pnpm check` runs the purity check, and it must pass.
- **Test first.** Write the test, see it fail for the right reason, then fix. Use the W3C structured shapes: a colour as `{colorSpace, components}`, a dimension as `{value, unit}`, a `fontFamily` as an array.
- **The decisions file** is added to, never rewritten. Do not write the round's entries: put what the build found in your final report, and the integration writes one entry per repository.
- **Which doc says what.** `reference.md` owns every shape and message. Change the doc that owns the statement: the guide for use, the specs for why, the reference for shape.
- **Write like the code around you.** Match its comment density, naming and error sentences.
- **Fix the root,** where every caller passes through it. **Do less:** the smallest diff that fixes the cause.
- **When a fix turns out to need a ruling,** stop on that item and say so in the report.
- **Commit messages** are one sentence in the house style, then a short body, ending with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.

## A. figma: the command line

Your files are `cli/`, the command line's tests, `test/support/fakeFigcraft*` and `fakeLibraries.ts`, and the sections of `reference.md`, `specs.md` and `guide.md` on the command line, the lock and FigCraft.

1. **A dry run finds the library by its id** (finding 8). `describeTarget` passes `listed` the id the target's lock holds for the library (`GET_LOCK`), and falls back to the path when the lock holds none. When FigCraft lists it at another path, the line says a run would point it there. Test it in the fake with a library enabled from a folder that has moved. The reference's dry-run lines (*Libraries in FigCraft*) say it.
2. **The lock's path, as question 1 rules.** `lockedFrom` and its two callers (`run.ts:411, 478`). An older entry that holds a path is still read. Update `reference.md:438`, specs *In FigCraft* and the guide's *A moved library* line.
3. **Paths in messages, as question 2 rules.** The library functions take the path for FigCraft and the label for the line apart. `Saved` prints by the same rule. The readers in `cli/bin.mjs:16` give Node's reason without its path, if the answer says so. The reference states the rule once, in *Headless* or the command line's opening, and the message rows follow it. Expect many message tests to change: change only the path in them.

## B. figma: the dry run's restack

Your files are `shared/components/stack.ts`, `main/components/endOfRun.ts`, `test/components/restack.test.ts`, and the restack sentences of `specs.md:1366`, `guide.md:725` and `reference.md:66, 95`.

1. **The dry run's line, as question 3 rules.** On a: `describeStack` takes its names from `restack()`'s moves. Test first: after a real run, a dry run says nothing of the page; a set moved by hand that a run would flow around still gets its line; a set made taller in the document is not promised a position.

## L. lib

Your files are `src/`, `test/` and lib's docs.

1. **The entry document is named once** in a read or parse failure (finding 2): `the entry document "<path>" could not be read.` Change the test that pins the double. A `$ref`'s message is unchanged.
2. **On question 4a: a token's custom property by its path.** A `DesignSystem` method, with the one-collection shorthand, that throws for a path naming no token and for an unknown collection in a system of two. It answers what `cssPropertyName(lookupPath(path))` answers. The reference owns its signature; the specs' questions table (*css*) lists it. Test it with one collection and with two, including a group named like the collection.
3. **On question 5a: `Check.unresolvable`.** Each token that throws `UnresolvableReferenceError` or `CircularReferenceError` when resolved in a mode: its path, the mode, the reference and the reason. Measure the time on a synthetic document of a thousand tokens and a product build of at least 27 modes, and report it. Test a colour `{colorSpace, components}` whose `$value` is a reference to nothing, a shadow member that refers to nothing, a reference to nothing in one context only, and a cycle. The reference's `Check` shape and `specs.md`'s *Future* line change with it.

## Q. query

Your files are `src/`, `sessions/`, `test/` and query's docs.

1. **The usage says the flag repeats** (finding 5): `--tokens <file>` given once for each file, in `src/main.ts:17` and `specs.md:48`. A test holds the usage line.
2. **`check` prints `sameCssName` and fails on it** (finding 9), under a header in the style of the others. A session with two dimension tokens of one name, `{value, unit}`. Every session that asks `check` gains the line. The specs' *Exit codes* and the guide's `check` row say it.
3. **Once lib's release is on npm:** move `@toghen/lib` to it, then the answers to questions 4 and 5. On 4a, `css <path>` calls the new export; on 5a, `check` prints `unresolvable` and fails on it. `a-broken-document.session`'s answer changes, and the diff is the proof, as it was for `applies`. On 4b, the `css` form names both exports in its last line.

## H. house, only on question 4b

The constitution's query rule gains the fourth form that reads two exports, a minor version, with Egildo's sign-off. The integration copies it into every repository.

## S. skills, after the releases

Every answer above changes something the skills say, and so do findings 5 and 9. Follow the routine in `skills/specs.md`, *Keeping a skill pinned*: read each release at its tag, measure the worked example again headless, update the pins, dates and plugin version, and add one dated entry to `skills/decisions.md`. The lines to look at first:

- `pipeline/messages.md:134, 205`; `pipeline/commands.md:78, 87, 92, 96, 187`; `pipeline/SKILL.md:134, 170, 284`; `pipeline/example.md:203, 261`; `figcraft/toghen-target.md:57`.

## Risks

- **query waits on lib.** Its worktree installs lib from npm, so items 3 and 4 cannot pass the gate until lib is published, which needs Egildo. figma, postcss and ui stay on lib 0.2 this round.
- **Question 3a can under-report.** A set the run makes larger may push others that the dry run did not name. The line must not claim a position.
- **Question 5a's cost** grows with tokens times modes. If the measurement is slow, say so before the integration, not after.
- **Golden answers move.** Message tests in figma and sessions in query change. A change to anything but the path or the new line is a regression.
- **FigCraft's link stays absolute** whatever the lock holds. A committed `.figcraft` file still changes when a run on another machine points the library at its folder.

## Not in this round

- **Paths given to lib relative.** figma hands lib absolute paths by a ruling (figma `decisions.md`, the command line sends the files themselves), so lib's messages name absolute locations.
- **query exits 2 when the document does not load, figma 1.** Both are what their docs say.
- **`type nope` answers `(none)`, exit 0.** lib's rule: `resolveToken()` returns `null` for a lookup miss.
- **FigCraft's own absolute link,** and what a run does where the library file is missing altogether, not moved. Neither was measured beyond the probe above.
- **Noticed, not looked into:** an unchanged run prints `Chip: 1 step · 0 created · 0 updated · 0 deleted`; a first headless library run names its target `"Untitled"` in the lock line.

## Integration, by Claude in the main session

1. **Review** each branch's diff against this plan and the rules. Send a unit back for what it missed.
2. **Merge** figma's two branches into `measurement-fixes`.
3. **Check across the packages.** Put each branch in the main checkouts, run `pnpm check` in each, rebuild the plugin, and confirm no `[object Object]` and no unresolved `toghen(`.
4. **Record** what each unit's build found in each repository's `decisions.md`, dated, one entry per repository. figma's entry supersedes the 23 October lock ruling where question 1 changes it, and closes the 23 November *Noticed* line on the restack.
5. **Report** to Egildo. Merging means pushing to Codeberg, on his word. Claude bumps the versions after the merge; a publish needs Egildo. lib goes first, then query's items 3 and 4, then figma and query together, then unit S.
