# Plan: a two-context colour modifier written with `light-dark()`

30 October 2026. A request from a consumer (generalised here): a theme modifier spelled as a media query lets only the
operating system choose the scheme; an app cannot offer its own switch. CSS's `light-dark(a, b)` follows the used
`color-scheme`, which an app sets on any element. postcss's `decisions.md` already keeps "a form per modifier" for
later, with the browser behaviour measured (`postcss/measurements/RESULTS.md`, case 5); this round designs and builds it.

Rulings (Egildo, 30 October 2026): 1a, 2a, 3a, 4a, stated below as requirements.

Repositories: `lib`, then `postcss`, both in `/Volumes/dev/Codeberg/Toghen`. The workspace links them, so postcss's
tests run against the local lib. Work on `main` in each; commit, do not push or tag.

## Read first

- `CLAUDE.md` in each repository, then `reference.md`, `specs.md` and the end of `decisions.md` in each.
- `lib/src/resolver/context-selector.ts` (`getContextSelector`), and where `lib/src/index.ts` exports it.
- `postcss/src/index.ts`: `contexts()` (around line 1117), the mixins writer (the `--when-` loop around line 1525),
  `spellings()` (around line 1560), `sameIn()`, `declarations()`.
- `postcss/test/contexts.test.ts`, `mixins.test.ts`, `spellings.test.ts`: the test style to copy.

## Rules that bind this work

- **Purity.** No consumer's name, palette or token paths anywhere: source, tests, docs, commit messages. Synthetic
  tokens only (`color.bg`, `color.line`, `shadow.raised`, …). `pnpm check` runs the purity check; it must pass.
- **Structured shapes.** Every colour in a test is the W3C shape `{ colorSpace: 'srgb', components: [...] }`; at least
  one test uses a dimension `{ value, unit }`.
- **Test first.** Write each test, run it, see it fail for the right reason, then implement.
- **Docs are added to, never rewritten**, in `decisions.md`. `reference.md` owns the shapes; other docs link to it.
- Match the surrounding code: its comment density, naming, error sentence style (`[@toghen/postcss] Cannot …`).

## 1. The document's shape (lib) — ruling 1a

A modifier's CSS member may carry `lightDark` instead of a selector or an at-rule:

```json
"$extensions": { "com.toghen": { "css": { "lightDark": { "light": "day", "dark": "night" } } } }
```

`light` names the context that is the first argument of `light-dark()`, `dark` the second.

Add to `lib/src/resolver/context-selector.ts`, and export from `lib/src/index.ts`:

```ts
export interface LightDark { light: string; dark: string }
export function getLightDark(modifier: { name?: string; contexts?: unknown; $extensions?: Record<string, any> }): LightDark | undefined;
```

- Returns `undefined` when there is no `css.lightDark`.
- `contexts` comes in two shapes: an object (a resolver document's modifier) or an array of names (a modifier a
  collection records, `collection.getModifiers()` / `resolver.getModifiers()`). Accept both: the names are the object's
  keys or the array's items. Check the real `ModeModifier` type in lib before writing this.
- Throws `ResolverValidationError`, in the style of `getContextSelector`'s messages, when:
  - `lightDark` is not an object, or `light` / `dark` is missing or not a string;
  - `light` equals `dark`;
  - either is not one of the modifier's contexts;
  - the modifier does not have exactly two contexts;
  - the same CSS member also has `selector`, `atRule` or `contexts` (two spellings for one modifier).
- `getContextSelector` itself stays as it is: for a `lightDark` modifier it returns `undefined` (there is no
  `selector`/`atRule`). Check that this is so, and add a test for it.
- Check whether lib validates the resolver document's `$extensions` anywhere else (load, schema); if it does, make
  `lightDark` pass there too.

Tests: `lib/test/unit/resolver/` (a new `light-dark.test.ts`, or beside an existing test of `getContextSelector` —
find it with grep). One test per thrown case, one for each `contexts` shape, one for "no field".

Docs (lib):
- `reference.md`: a `getLightDark` section beside `getContextSelector`'s, with the shape, return value and each error.
  `getContextSelector`'s section gains one sentence: a modifier with `lightDark` has no selector spelling.
- `decisions.md`: a dated entry, "A two-context colour modifier spelled with `light-dark()` (30 October 2026)": the need,
  the shape, the rejected alternative (a per-context `colorScheme` under `contexts`: longer, and it does not say the
  two contexts are a pair).

Bump lib to `0.1.1` (`package.json`, and any other version constant `grep -rn "0\.1\.0" src` finds). `pnpm check`
green. Commit: "Read a modifier spelled with light-dark()".

## 2. Writing the contexts (postcss) — rulings 2a and 4a

In `contexts()`, for a modifier where `getLightDark(modifier)` returns a pair:

- **No context blocks.** Neither context gets a block, a sublayer of its own, or an error for having no spelling. The
  layer order statement keeps the modifier's name anyway (so an order shared across directives does not change);
  check what an empty sublayer does today — `sublayer()` writes nothing for no nodes — and keep that.
- **The pair goes in the defaults.** For each declaration this modifier varies (the same `varied` test the block path
  uses), the declaration under the root selector becomes `light-dark(<value in the light context>, <value in the dark
  context>)`. Get the two values the way the block path does: `declarations(…)` with the mode
  `collection.modeNamed(\`${modifier.name}=${context}\`)` and the `varied` filter, then match by `prop`. A colour that
  does not differ between the two stays a plain value (it is not varied, so nothing changes).
- **Only colours** (ruling 2a). If a varied declaration's token is not `$type: 'color'`
  (`system.resolveToken(token, mode)?.$type`), report an error and write nothing for that modifier:
  `[@toghen/postcss] Cannot write modifier '<modifier>' with light-dark(): '<token>' is a <type>, and light-dark() takes only colours. Write the modifier with a selector or an at-rule, or vary only colours in it.`
  Use `report(atRule, result, …)` as the clash error does, so `onUnresolved` decides whether it throws.
  Under `references: 'var'`, a non-colour token that only reads a colour token (a shadow reading `var(--color-shadow)`)
  is the same text in both contexts, so it is not varied and needs no pair. Prove this with a test; do not special-case it.
- **Another modifier on the same token.** If a token this modifier varies is also varied by another modifier, the other
  modifier's block would replace the pair with a plain value and drop the scheme. Report an error naming both
  modifiers and the token. Mark it `// ponytail: …` as a known ceiling (a pair per block would lift it).
- **Chains.** With `bind: 'chain'`, a `lightDark` modifier is an error: `[@toghen/postcss] Cannot chain modifier '<modifier>': it is written with light-dark(), which has no blocks to chain. Leave out bind: 'chain', or spell the modifier with a selector or an at-rule.`
- **`registerProperties`.** A registered `<color>` property may hold `light-dark()` as its value; its `initial-value`
  stays the light literal (`registrations()` reads the default mode). Check `registrations()` still runs on the
  rewritten declarations and add one test.
- **`color-scheme` is not written** (ruling 4a). The app sets it. Without it the used scheme is light and every pair
  shows its light side; the guide says so.
- **The defaults' comments** (`descriptions: 'comment'`): keep whatever `commented()` writes for the defaults; the
  modifier's own description, which the blocks carried, goes before the root block's first pair — or is dropped if
  that is awkward; say which in the reference.

## 3. Mixins and spellings (postcss) — ruling 3a

A `lightDark` modifier has no `--when-<modifier>-<context>` mixin: CSS cannot ask "is the used scheme dark?", and a
mixin that set `color-scheme` would change what the mixin means.

- In the mixins writer, skip such a modifier and write one comment in its place, in the style `spellings()` uses for
  what it leaves out: ` --when-<modifier>-<context> is left out: modifier '<modifier>' is written with light-dark(); set color-scheme on the element instead. `
- In `spellings()`, the same: no `:--<modifier>-<context>` and no `@custom-media`, one comment per context saying why.
  Today `getContextSelector` returns `undefined` so the loop would skip silently; add the comment explicitly.

## 4. Tests (postcss)

A new `postcss/test/light-dark.test.ts`, its document built like `contexts.test.ts`'s: a `theme` modifier with
`lightDark` and contexts `day` / `night` (not `light` / `dark`, so the test proves the names are read from the field),
and a second modifier `density` spelled by attribute. Cases:

1. The root block holds `light-dark(<day>, <night>)` for each colour that differs; a colour equal in both is plain; no
   block or `@media` appears for the theme.
2. `references: 'var'`: a colour that references another colour writes `light-dark(var(--a), var(--b))` or whatever
   the block path would have written on each side — assert the exact output.
3. A shadow (structured `{ color, offsetX: {value,unit}, … }`) that reads a themed colour, under `references: 'var'`:
   no error, no pair for the shadow.
4. The same shadow under `references: 'resolve'`: the colour-only error.
5. A dimension varied by the theme: the colour-only error.
6. A colour varied by both `theme` and `density`: the two-modifier error.
7. `bind: 'chain'`: the chain error.
8. Inside `@layer`: the order statement still names `theme`; no `@layer theme` block.
9. `@toghen mixins`: no `--when-theme-day` / `--when-theme-night`; the comment is there; `density`'s mixins unchanged.
10. `@toghen spellings`: no theme spellings; the comments are there.
11. `registerProperties`: the `@property` for a paired colour has the day literal as `initial-value`.
12. No `[object Object]` and no unresolved `toghen(` in any output above (assert on the whole CSS).

Every existing test stays green unchanged.

## 5. Docs (postcss)

- `reference.md`:
  - The section on `@toghen contexts` (around line 229): a paragraph on a `lightDark` modifier — no blocks, pairs in the
    defaults, colours only, and link to lib's `getLightDark` for the shape rather than restating it.
  - The mixins table (around line 253) and the spellings table (around line 306): one line each, "not for a modifier
    written with `light-dark()`".
  - The errors table (around line 453): the three new errors, with cause and fix, in the table's columns.
- `guide.md`: a short section, "Letting the app choose the scheme": spell the theme with `lightDark`, then
  `:root { color-scheme: light dark; }` follows the system, `color-scheme: light` or `dark` on any element chooses, and
  without any `color-scheme` the light side shows. A media-query spelling remains the choice for browsers older than 2024.
- `specs.md`: if it has a section on how contexts are written, one paragraph on this form and why it has no blocks.
- `decisions.md`:
  - A dated entry, "A two-context colour modifier written with `light-dark()` (30 October 2026)": the rulings 2a, 3a, 4a
    with the alternatives rejected (2b: warn and write the light value; 3b: a mixin that sets `color-scheme`; 4b: the
    plugin writes `color-scheme: light dark`, which would override the document's default context), and the
    two-modifier ceiling.
  - The open point "A form per modifier" (around line 841): prefix it with `*Closed 30 October 2026.*` and a pointer to
    the new entry, as the file does with "*Moot since …*" elsewhere. Do not rewrite its text.

Bump postcss to `0.1.1`, its `@toghen/lib` dependency (both places in `package.json`) to `^0.1.1`. Run `pnpm install`
at `/Volumes/dev/Codeberg/Toghen` if the lockfile needs it. `pnpm check` green in postcss. Commit: "Write a two-context
colour modifier with light-dark()".

## 6. Done means

- `pnpm check` green in `lib` and in `postcss`.
- `pnpm --filter @toghen/figma build` still succeeds (its stylesheet goes through postcss; nothing there uses
  `lightDark`, so its output must not change — diff the built stylesheet before and after).
- Two commits on `main`, one per repository, each ending with
  `Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>`. Nothing pushed, nothing tagged.
- A short report: what was built, any place the plan was wrong or silent and what was decided there, and anything
  left undone.

## Out of scope

- Writing `color-scheme` anywhere.
- A modifier with more than two contexts, or a pair for non-colour types.
- The figma adapter: it reads modifiers as Figma modes and does not read CSS spellings. Check with
  `grep -rn "getContextSelector" /Volumes/dev/Codeberg/Toghen/figma/{main,shared,cli}`; if it does read them, stop and
  report instead of changing figma.
