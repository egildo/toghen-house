# Plan: the obvious fixes from the review of 23 November 2026

## Where this comes from

This round fixes what the review of 23 November 2026 found, where the fix is clear. The review is `review-2026-11-23.md`, beside this file. Each finding there was proved, either by a probe or by quoting the lines that show it. The probes are kept under `/private/tmp/claude-501/-Users-egildo--claude-spares-spare-Fpfl06/85778d0b-8740-48e9-8b29-268e30e7de4d/scratchpad/review-*/`. They may be gone by the time you read this. If one is, read the code and write your own probe.

Egildo's word, 23 November 2026: "Plan all the obvious fixes and send opus agents to code."

**Obvious** means two things:

- the defect is proved;
- the fix follows from a rule Toghen already has, whether in the constitution, a package's docs, the W3C specs or the package's own stated intent.

Anything that needs a new ruling is listed under *Not in this round*, and it stays out.

## The repositories, the branches and the worktrees

- **The checkouts.** The repositories are the checkouts in `/Volumes/dev/Codeberg/Toghen`.
- **One worktree per unit.** Each unit works in its own git worktree under `/Volumes/dev/Codeberg/Toghen-worktrees/`. The folder sits outside `/Volumes/dev/Codeberg/Toghen` on purpose: that folder's `pnpm-workspace.yaml` would capture a nested worktree.
- **Branches.** Each branch is cut from the repository's `main`; gosfuig's is cut from `origin/main` (see unit I).
- **Commits.** Commit on the branch. Never push, tag, publish or merge, and never touch `main`.
- **Installing.** Install inside the worktree with `pnpm install --frozen-lockfile`, or `npm ci` for gosfuig. The packages resolve their neighbours from npm there, which is what CI does.
- **The gate.** Each repository's own `pnpm check`, run inside the worktree, is the unit's gate. gosfuig's is `npm run check`. A unit is done only when its gate is green.

| Unit | Repository | Worktree folder | Branch |
|---|---|---|---|
| A | figma: command line and FigCraft | `figma-cli` | `review-fixes-cli` |
| B | figma: engines | `figma-engines` | `review-fixes-engines` |
| C | figma: screen, and ui | `figma-screen`, `ui` | `review-fixes-screen` (figma), `review-fixes` (ui) |
| E | lib | `lib` | `review-fixes` |
| F | postcss | `postcss` | `review-fixes` |
| G | query | `query` | `review-fixes` |
| H | house and skills | `house`, `skills` | `review-fixes` |
| I | gosfuig | `gosfuig` | `review-fixes` |

## Rules that bind every unit

- **Read first.** Read the constitution (`.specify/memory/constitution.md`), the repository's `CLAUDE.md`, and the docs of the package you change. Read them before you change it. Touching the plugin's screen means reading `ux-rulebook.md` first.
- **Purity.** No consumer's name, palette, token paths or components, anywhere: source, tests, docs or commit messages. Use synthetic tokens only. `pnpm check` runs the purity check, and it must pass.
- **Test first.** Write the test, see it fail for the right reason, then fix. Use the W3C structured shapes: a colour as `{colorSpace, components}`, a dimension as `{value, unit}`, a `fontFamily` as an array.
- **The decisions file.** `decisions.md` is added to, never rewritten. Removing a consumer's name, in unit A, is the one edit in place this round allows. Do not write the round's entries yourself: put what the build found in your final report, and the integration folds it in, one entry per repository.
- **Which doc says what.** `reference.md` owns every shape. When behaviour changes, change the doc that owns the statement: the guide for use, the specs for why, the reference for shape.
- **Write like the code around you.** Match its comment density, naming and error sentences.
- **Fix the root.** Fix the cause where every caller passes through it.
- **Do less.** Take the smallest diff that fixes the cause. Add no abstraction that nothing needs.
- **When a fix turns out to need a ruling,** stop on that item and say so in the report. Do not guess.
- **Commit messages** are one sentence in the house style ("Read the stamps without a depth, which figcraft-cli now refuses"), then a short body. End each message with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.

## A. figma: the command line and FigCraft

Your files are `cli/`, `test/support/fakeFigcraft*`, the command line's tests, `ui/bridge.js`, `build.mjs`, `scripts/`, `package.json` (scripts only), `shared/version.ts`, and the sections of `guide.md` and `reference.md` on the command line and FigCraft.

1. **Turn main green.**
   - Format `test/support/fakeFigcraft.ts`.
   - Make `shared/version.ts` say `0.1.16`, as `package.json` does.
   - Make a hand publish run the gate: `prepack` runs `pnpm check`, or the equivalent, so a red tree cannot be packed.
2. **Library mode headless.** `pinPath`, `savePinned`, the guard at `run.ts:362` and the closing at `run.ts:430` all find a tab by `doc.list`'s `path`. FigCraft leaves `path` out for MCP calls (its v0.18.1 `docs/control-protocol.md`). Track the tab by what FigCraft does return: `id`, `index` and `name`, and what `open_file` and `file.new` answer. It must work both headless and with `--figcraft <port>`.
3. **The fake behaves like FigCraft 0.18.1.**
   - It omits `path` from `doc.list` when the call names an agent.
   - It refuses an unknown parameter on every command.
   - Its `save_file` fails when the folder is missing.
   - Its default version is 0.18.1.
   - Then make sure the library tests cover item 2.
4. **`INIT_CWD` (`cli/bin.mjs:14`).** The installed bin resolves paths against `process.cwd()`. If the package's own `toghen-figma` script needs the folder where it was typed, that script passes it explicitly. Add a test where a script changes directory first.
5. **Nothing hangs.**
   - When a plugin window is replaced, the pending request rejects with a sentence, and the UI's bridge does not retry on `REPLACED`.
   - A call after FigCraft's socket closes rejects.
   - The headless session times out `initialize` and each request.
   - A non-JSON line on stdout is reported, not a crash.
   - Replies are matched by `id`.
6. **Success is printed after the save it reports** (`cli/figcraftLibrary.ts:86`, `93`).
7. **A missing folder for `--figcraft-file` is refused before the sync starts.**
8. **The docs.**
   - Add `--skip-unchanged` and `--prune` to `reference.md`'s options.
   - Correct its line saying `node.get` takes a depth.
9. **Nits.**
   - `build:watch` watches.
   - The preview server listens on localhost only.
   - `toghen-figma --help` prints the usage and exits 0.
10. **Purity.** In `decisions.md:1725` and `.reference/plans/plan-2026-10-22-figcraft-components.md:96, 100`, replace each downstream project's name with "a consumer". Grep the whole repository for any other.

## B. figma: the engines and shared code

Your files are `main/`, `shared/` (except `version.ts`), `cli/run.ts` only where a dry run's counts or the style plan's input need it, their tests, and the protocol and shape sections of `reference.md` and `specs.md`.

1. **Paint binding.** `bindPaintColor` (`main/engines/operationExecutor.ts:165`) gives the paint the variable's resolved colour before binding, as `fieldWriter.ts:136` does. Measured behaviour §8 is the reason.
2. **A set or artboard update that fails partway** stamps every write it made and does not keep the old fingerprint (`main/components/update.ts:255`, `run.ts:166-169, 239`). A later `--skip-unchanged` reads it, and a full run repairs it rather than calling it drift.
3. **A new artboard that fails** removes what it made, as a new set does (`main/components/apply.ts:664-689`, `orRemoveWhatItMade`).
4. **`producedSymbol`** (`main/engines/batchRunner.ts:172`) counts `ADD_MODE` and the `$mode:` symbols of `CREATE_COLLECTION`. A refused mode is then one failure, and its values are "prerequisite failed".
5. **Commands run one at a time** on the main thread (`main/dispatcher.ts`). A plan that arrives mid-execute is not wiped (`executeVariableSync.ts:38-58`).
6. **A typography token's literal `fontWeight`** that the style cannot carry gives a warning that names it, instead of a silent "Regular" (`main/utils/figmaStyleAdapter.ts:187`). Do not map weights to style names: that is a ruling.
7. **The fingerprint names a library component by its key,** not its node id (`main/components/fingerprint.ts:73`, measured behaviour §15).
8. **The dry run's counts.**
   - A component set's dry run reports the creates and deletes the real run would make (`cli/run.ts:670`, `main/components/run.ts:186-193`).
   - A dry-run style plan is told the variables the same run would create (`cli/run.ts:580`), so it gives no "sync variables first".
9. **`reference.md`** gains what the shapes and the protocol have and it lacks:
   - `RunOptions.skipUnchanged`;
   - `Pending.removed` and `Pending.components`;
   - `Report.artboard`;
   - `RUN_COMPONENT`'s `inputs`;
   - the `COMPONENTS_CHECKED` row;
   - the document state's fields;
   - `FigmaCollectionPayload.variableCount`;
   - `PlanCounts`.
   
   Correct the `FINISH_COMPONENTS` comment in `protocol.ts`.
10. **Tests.**
    - Run `skipUnchanged` against `figmaMock`, including a failed run.
    - Add a W3C typography token without `fontStyle`.

## C. figma: the screen, and ui

Your files are `figma/ui/` (except `bridge.js`), `figma/ui.md` and `figma/ux-smell-log.md`, and in the ui repository `src/`, `tokens/` (structure only, not colour values), `package.json`, `specs.md` and `guide.md`. Read `ux-rulebook.md` first and do not edit it.

1. **Script injection.**
   - `status-bar.js:33` uses `escAttr` for the `title`.
   - Grep every attribute built from text for the same mistake.
   - `popover.js`'s `aria-label` is built from unescaped text, and escaped once.
2. **R11, the drift note.** The note may give an all-clear only from a check that covered the tokens it speaks of. Where the style plan and the Tokens look disagree, the stronger answer wins (`pane-sync.js:28-41, 103-106`).
3. **R16, the collection dot.** It reads every state the Tokens pane reads, `adopted` and `blocked` included, the same way (`pane-sync.js:48-57` against `pane-tokens.js:45-60`).
4. **Cancel.** The screen stays busy, with Execute off, until `OPERATION_COMPLETE` (`results.js:370-373`).
5. **Accessibility.**
   - `patch()` puts focus back on the control that had it (`base.js:13, 62-67`).
   - The result bar's toggle is a real focusable button, with the chips not nested inside it.
   - The splitter is a `role="separator"` that the arrow keys move.
   - The popover takes focus when it opens and gives it back when it closes.
   - In ui, a row's buttons are not inside a `role="option"`; follow the fix of ui `decisions.md` #4 for disclosures.
6. **`color-scheme`** is declared, so native controls follow the theme. Put it where the theme is chosen, in ui if that is ui's job.
7. **ui packaging.**
   - Remove the `./build.mjs` export and `scripts/` from `files`. First check that nothing imports them: figma's `build.mjs` reads only `@toghen/ui/styles.css`.
   - Move `@phosphor-icons/web` to devDependencies if only the build reads it.
   - Correct `specs.md:57-59`.
8. **Docs.**
   - `ui.md`: name the files and elements that exist; `data-state`; eight states; two `<style>`s; three fonts.
   - Smell log: mark #8 fixed, and make the header agree with #51's row.
   - Leave R8's wording, and every rule, to Egildo.
9. **Tests.** Run the screen's pure parts in Node under a small DOM shim: `resultView`, `driftKnownFrom` and each element's `update()`. Cover items 1 to 4.

## E. lib

Your files are `src/`, `test/` and lib's docs.

1. **`color-mix` at 0% or 100%** with a translucent colour returns that colour exactly (`src/utils/css-color.ts:212`).
2. **A structured colour interpolated into a string** is spelled from its `colorSpace` and `components`, never from its `hex`, which is a fallback (`src/core/alias.ts:389`, `src/utils/css-color.ts:287-299`).
3. **Overrides beside a `$ref` in `resolutionOrder`** are honoured everywhere `resolve()` honours them: mode names, `getModifiers`, `setModeContexts`, the sum build and the chain index (`resolver.ts:549-558, 713-728, 796-803`). This is a W3C MUST.
4. **Modifier names and context names** match the way resolver input matches: without regard to case, as the document spells them (`resolver.ts:343, 385-393`).
5. **Prototype keys.** Use null-prototype objects or `Object.hasOwn` in the token tree, the vocabulary and the merge, so that `__proto__`, `constructor` and `toString` are ordinary names (`collection.ts:439-455, 566-573, 877`, `resolver.ts:104`).
6. **Safe sharing, as `specs.md` promises.** `resolve()` returns values not shared with the document. `DesignToken` does not keep `$extensions`, or a `$value`, by reference. Freezing them or copying them are both acceptable; measure the cost, and say which you chose.
7. **A JSON Pointer `$ref` as a token value** throws a named error saying it is not supported yet (`alias.ts:192-198`).
8. **`cssPropertyName`** escapes what is not a valid identifier, as `cssLinkName` does. Two tokens that would get one custom-property name are reported by `check()`.
9. **A set whose sources refer to itself** throws a named error in `Resolver` (`resolver.ts:877-920`).
10. **A gradient stop outside 0 to 1 is clamped,** as W3C §9.7 says. First check `decisions.md` for a ruling to refuse it; if there is one, leave it and say so.
11. **`fontFamily` is checked at load:** a non-empty string, or an array of non-empty strings. A reference to a stack inside an array is flattened.
12. **Docs.** Fix the guide's first example and the JSDoc that names removed APIs; see the review, lib item 13, for the list. Fix `reference.md` on `getToken('default')`, and the nine-questions table (`getLightDark`, `literalText`).
13. **Tests.**
    - Resolver tests in the structured shapes.
    - The factoring property test asserts its `checked` count.
14. **Nits.**
    - Delete the two stray `.test.d.ts` files.
    - `setToken` refuses empty segments.
    - `getGroupAttribute` checks the mode.

The merge rule is not in this round. A context's colour still merges key by key; see *Not in this round*.

## F. postcss

Your files are `src/`, `test/` and postcss's docs.

1. **One member with no CSS spelling no longer hides the whole mixin silently.**
   - Report the token and its shape, as `specs.md:347` says.
   - Throw under `onUnresolved: 'throw'`.
   - Write the rest of the mixin, as an error in one group writes every other group.
   - Correct `reference.md:425`.
2. **A `light-dark()` pair is not registered as `<color>`** under `registerProperties`, so `color-scheme` on a subtree still switches it (`src/index.ts:1211-1216`). Correct the test at `test/light-dark.test.ts:167`.
3. **Layer names go through `ident()`,** as mixin names do (`:1139-1144, 1308`).
4. **A token defined on one side only of a `light-dark()` modifier** gives a warning that names it (`:1195`).
5. **Each failure is reported once, with its position** (`:548-551, 765`).
6. **`t(` and `toghen(` inside strings and `url()` are left alone** (`:429, 432`).
7. **A `t()` or `toghen()` in an at-rule's parameters is reported,** so `'throw'` fails the build on it.
8. **Generated nodes carry the `source` of the directive** they came from.
9. **The `light-dark()` refusal names the member's type,** not "value" (`:1197, 1203`).
10. **Remove the monorepo leftovers.**
    - Fix the specs' directory list and build commands.
    - Remove the `toghen-postcss/specs.md` path from the error at `:158`.
11. **Make the example palette names generic** (`specs.md:506`, `test/fallback.test.ts:21-22`), as `reference.md:202` already did.

## G. query

Your files are `src/`, `sessions/`, `test/` and query's docs.

1. **A printed string is never a blank line, and never a line a session reads as a comment.** Escape a newline and a leading `#`, and keep every existing session's answer unchanged (`src/print.ts:11`).
2. **`css <--property> <modifier>` runs as a one-shot query.** A usage error exits 1, as the specs say (`src/main.ts:21-32`).
3. **`value <path> across`** prints a row per mode, with an error on the row that throws (`src/commands.ts:137`).
4. **Refuse three silent cases.**
   - `across` with a pair.
   - A query given with `--session`.
   - `pairsOf` splits at the first `=`.
5. **The guide.**
   - The collection form of a path works only for value, type and refs; say so.
   - Replace the monorepo commands with the commands of this repository.
6. **Add a `prepare` script, and a short `README.md`** if none is tracked. Check first that one is not ignored.
7. **Sessions** that answer with a multi-line string and with a line starting `#`.

## H. house and skills

Your files are house's `shared/`, `claude-md/`, `speckit-preset/`, `README.md` and `.reference/plans/workspace/plan-2026-10-30-light-dark.md`, and the whole of skills. The integration copies house's shared files into each repository, so do not edit the other repositories.

1. **`publish.yml`.**
   - A check job without `id-token`, then a publish job that holds `id-token: write` and only publishes.
   - Before the check, it fails at once when the version is already on npm.
2. **Actions.**
   - `pnpm/action-setup` goes to a release that runs on Node 24, if one exists; check upstream.
   - Actions are pinned by SHA, with the tag in a comment.
   - Change `check.yml` the same way.
3. **The purity check fails closed under CI.** When `CI` is set and there is no names file, or the file is empty, it exits 1.
4. **The purity script.**
   - Format house's copy in Biome's style, so every copy is identical to it.
   - Rename the `escape` helper, so the `noShadowRestrictedNames` overrides can go. List which repositories' `biome.jsonc` lose that override, for the integration.
5. **`claude-md/*.md`, `house.md` and `README.md`.**
   - Use the real gate commands: `pnpm check` for the five code packages, and the purity script for house and skills.
   - Nothing "still to come" that exists.
   - The package table includes skills and gosfuig.
   - No `toghen-figma/` paths.
   - Node 24, as `.node-version` says.
   - Leave the line on CLAUDE.md tracking as it is; that is a ruling.
6. **The preset.**
   - The house checklist replaces the stock "no implementation details".
   - The Answers date has its own placeholder, which survives "delete every comment block".
7. **Commit the light-dark plan,** which shipped on 30 October.
8. **skills.**
   - The install line is the live marketplace. Check what `egildo/toghen-skills` serves.
   - The setup is the npm setup of the figma guide.
   - The version pins are figma's current version and FigCraft 0.18.1.
   - The `messages.md` lines are what `cli/run.ts` prints today, `· N drifted` included.
   - Check every command and flag a skill names against `figma/cli/main.ts`'s usage and `figcraft-cli commands --json`.
   - Add a dated line to `skills/decisions.md` correcting its Biome and `pnpm check` claim. Do not rewrite the old line.
   - Add a short `README.md` if none is tracked.

## I. gosfuig

Your files are the whole of gosfuig. Cut the branch from `origin/main`: the local main has an unpushed commit that untracks `CLAUDE.md`, and that is a ruling. Read `CLAUDE.md` first. Its *Known defects, deliberately not fixed* are pinned by tests and stay pinned. If a fix here would move one, stop on it and report.

1. **An achromatic pin or seed has no hue.**
   - Routing keeps the family's hue for it (`src/family.ts:279, 290`, `src/color/oklch.ts:32`).
   - A gray seed gives a neutral family, with no tinted ends, and a chroma peak that is never negative (`family.ts:411-415`, `curves.ts:107-111`).
2. **`bumpPeakFor`** does not ask for a chroma sRGB cannot hold near the light end (`curves.ts:106`).
3. **Step collisions are refused at validation,** as the README says: a colour and a lock on one step, one step named both ways, and more colours than steps (`src/spec.ts:139-229`).
4. **`wcag()` gamut-maps a string input,** as `apca()` does (`src/color/contrast.ts:18, 24`).
5. **DTCG export.**
   - Refuse `.` as a separator, and fix the test that asserts it.
   - Refuse a name starting with `$`.
   - Wrap a hue rounded to 360 to 0.
   - `colorSpace: 'srgb'` emits the object form `{colorSpace, components, hex}`.
   - Correct the README.
6. **A step count that cannot be named uniquely is refused** (`src/naming.ts:64`).
7. **An input with alpha below 1 is refused with a sentence,** not silently made opaque.
8. **Hue offsets between two pins go the short way between them** (`family.ts:296-313`).
9. **The README.**
   - The cyan example's numbers.
   - Six template fields.
   - What `slug: "none"` does.
   - What `check` runs.
10. **Packaging.**
    - The two font packages go to devDependencies.
    - Add a `default` condition beside `import`.
11. **The dashboard reads the step count and K, instead of fixing them at 12 and 4** (`src/dashboard/render.ts:54, 149`).
12. **Tests.**
    - Reference values for the conversions: `#ff0000` gives L 0.627955, C 0.25768, H 29.234.
    - Reference values for APCA: 63.06 for `#888` on `#fff`.
    - Gray pins and seeds, DTCG ranges, and the legality of names.

## Not in this round: rulings for Egildo

- **The lib merge rule.** Replace a `$value` whole, or merge but treat a colour as one value.
- **R15 against the inspection cleared after every operation.**
- **R14 against ui.md's chip rule,** for a drifted SKIP.
- **R3, for the build's checkbox and Execute in the result surface.**
- **Abandoned variables and styles.** Either the sync removes them behind the gate, or `--prune` covers them. The prune spec's assumption is false either way.
- **The R8 rename,** Selection to Apply.
- **Whether `CLAUDE.md` is tracked.** The `.gitignore` files against the constitution; gosfuig's unpushed commit.
- **The ui palette's contrast below 4.5:1** for `tg.text.3`, `text.on-accent` and `status.ok`. It changes the look.
- **Mapping a `fontWeight` to a Figma style name.**
- **ui's 23 tokens with `$type: "string"`,** and the lib extensions they rely on.
- **With `--figcraft --use`, the lock is written before the check.** It is documented as first.
- **Adding the two names to the purity list and the mirrors' `PURITY_NAMES` secret.** This writes outside the repositories.
- **The npm versions the old monorepo already holds.** The publish guard of unit H makes a collision fail early; which number to skip to is a release choice.

## Not in this round: measured first

- A text style whose `fontWeight` is a reference, rewritten on every sync: Figma's text-style behaviour is unmeasured.
- A translucent colour on a paint, possibly given its alpha twice.
- Clipping colours out of the gamut without a report waits for lib's gamut question.

## Integration, by Claude in the main session

1. **Review** each branch's diff against this plan and the rules above. Send a unit back for what it missed.
2. **Copy** house's shared files from unit H into each repository's branch: the workflows, the purity script, and the `biome.jsonc` overrides that go. Then copy the `claude-md` sources into the local `CLAUDE.md` files.
3. **Merge** figma's three branches into `review-fixes`.
4. **Check across the packages.**
   - Put each repository's branch in the main checkouts, so the local workspace links lib, postcss, ui and figma at this round's code.
   - Run `pnpm check` in each.
   - Rebuild the plugin, and confirm no `[object Object]` and no unresolved `toghen(`.
5. **Record** what each unit's build found in each repository's `decisions.md`, dated, one entry per repository.
6. **Report** to Egildo. Merging means pushing to Codeberg, on his word. Versions are bumped by Claude after the merge, and a publish needs Egildo.
