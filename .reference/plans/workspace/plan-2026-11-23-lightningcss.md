# Plan: the switch to Lightning CSS

**Created**: 23 November 2026

**Status**: Draft. The questions are open, and nothing is built until they are answered.

**Input**: Egildo, 23 November 2026: "Plan the switch to Lightning CSS."

Lightning CSS is a CSS parser, transformer, bundler and minifier written in Rust by the Parcel team (Devon Govett), published on npm as `lightningcss`. This plan says what it is today, what it would mean for `@toghen/postcss` and for the two builds that run it (ui's stylesheet, the Figma plugin), what the options are, and which questions are Egildo's.

**How this was found.** The repositories were read at their `main` checkouts in `/Volumes/dev/Codeberg/Toghen/`. The research used primary sources: Lightning CSS's own types (`node/index.d.ts` and `node/ast.d.ts` of 1.33.0), its docs and issues, npm, Vite's published code, and the specs and release notes named beside each fact. Sixteen small experiments ("spikes") were run against Lightning CSS 1.33.0, its WASM build and, for one check, 1.32.0, in the scratch folder `/private/tmp/claude-501/-Users-egildo--claude-spares-spare-Fpfl06/85778d0b-8740-48e9-8b29-268e30e7de4d/scratchpad/lightningcss/spikes/`. Some spikes ran the checked-out `@toghen/postcss` build and ui's build functions, and those read the repositories without writing to them. The scratch folder may be gone by the time this is read. Each spike is described well enough here to write it again.

Anything not confirmed from a primary source or a spike is marked **unconfirmed**.

---

## Summary

- **Lightning CSS cannot expand mixins.** It keeps `@mixin` and `@apply` in place as unknown rules, but nothing in a pipeline without PostCSS turns `@apply` into declarations. The mixins are how Toghen writes components. Only Chromium parses `@mixin`, and only behind a flag.
- **Lightning CSS corrupts token paths if it runs before Toghen.** Its tokenizer reads `.500` in `t(color.gray.500)` as the number 0.5, and prints `t(color.gray.5)`. `t(tg.sp.0)` comes out as `t(tg.sp0)`, which failed ui's real build in the spike.
- **Consumers already run Lightning CSS after Toghen, and it breaks `light-dark()`.** Vite 8 minifies CSS with Lightning CSS by default, at targets older than `light-dark()` support. Lightning CSS rewrites Toghen's `light-dark()` pairs into a form that is invalid unless `color-scheme` is declared in the same stylesheet. Even then it cannot follow a subtree that switches scheme.
- **Lightning CSS's JS extension API has an open regression on 1.33.0.** A visitor that hands back any rule holding a `var()` fails with "failed to deserialize". That happens even when the visitor returns the stylesheet it was given unchanged.
- **Speed is not a reason to switch.** On ui's real stylesheet, about 350 ms of the 354 ms build is Toghen's own work. PostCSS parsing and printing take about 3.5 ms; Lightning CSS's take 1.4 ms.
- **Minifying saves little.** It would take about 15 KB off a 436 KB stylesheet, most of which is embedded fonts. esbuild, which figma already has, comes within 2 KB of Lightning CSS.
- **A second adapter is feasible.** A thin Lightning CSS layer over a core that writes CSS text gave byte-identical output on the running example. The text is parsed back into Lightning CSS's tree.

**The recommendation:**

- Do not move the adapter now. Keep `@toghen/postcss` on PostCSS, and treat Lightning CSS as an environment Toghen's output must survive.
- Leave the builds as they are.
- Build a second adapter only when a consumer needs a pipeline without PostCSS, the visitor regression is fixed in a release, and the mixins and path questions below are answered.

The questions start with what "the switch" should mean.

---

## Constitution check

| Rule | What this plan does with it |
|---|---|
| The purity rule | No consumer is named. The spikes use lib's running example and ui's own tokens. The prior art names public tools, not consumers. |
| The core owns the semantics | Untouched by the recommended option. Moving CSS value spelling into lib would collide with lib's rule "It writes nothing out: an adapter serialises" (`lib/specs.md:11`). That is listed as an option and needs an amendment. |
| Adapters translate | Every option keeps translation in an adapter. |
| The invariants: "tsup builds lib and postcss; esbuild builds figma" | Unchanged unless a build adopts Lightning CSS. |
| Tests prove the documented format | The parity test runs over the golden files, which are written from lib's example in the W3C structured shapes. |
| postcss: "It emits standard CSS and no dialect of its own" | Kept. A quoted path `t("a.b.500")` would be a new form of the plugin's input, not of its output. |
| postcss: "A warning goes through the `result` the visitor is given" | This rule is PostCSS's. A Lightning CSS adapter has no `result`, and would need a rule of its own: an amendment, minor version. |
| postcss: "On a failure that does not throw, the original `toghen(...)` call stays in the output" | It can hold in Lightning CSS only if the directives are read as unknown rules. A directive declared as a custom at-rule and left in place is printed without its `;`, and the minified sheet breaks (spike 6). |

---

## What Lightning CSS is, today

### Versions and maintenance

- **Latest:** 1.33.0, released 20 July 2026 ([npm](https://registry.npmjs.org/lightningcss)). `lightningcss-wasm` is released in step with it, at the same version and timestamp.
- **Release dates since 2025:**

  | Version | Date |
  |---|---|
  | 1.29.0 | 9 Jan 2025 |
  | 1.29.2 | 6 Mar 2025 |
  | 1.29.3 | 14 Mar 2025 |
  | 1.30.0 | 11 May 2025 |
  | 1.30.1 | 14 May 2025 |
  | 1.30.2 | 29 Sep 2025 |
  | 1.31.0 | 20 Jan 2026 |
  | 1.31.1 | 21 Jan 2026 |
  | 1.32.0 | 9 Mar 2026 |
  | 1.33.0 | 20 Jul 2026 |

- **No release for four months.** Nothing has been released since 20 July, though commits were still landing on 22 November.
- **Open work:** about 338 open issues and 80 open pull requests (GitHub search, 23 November 2026).
- **Maintainers:** Devon Govett wrote 62 of the 141 commits since January 2025 ([commits](https://api.github.com/repos/parcel-bundler/lightningcss/commits)).
- **The Rust crate is pre-1.0:** `1.0.0-alpha.72` ([crates.io](https://crates.io/crates/lightningcss)).
- **API stability:** no statement found on a 2.0 or on the stability of the JS visitor API (**unconfirmed**). The current types and docs do not mark the visitor API experimental.
- **Licence:** MPL-2.0. It is copyleft by file, so it is fine as a dependency of MIT packages. Toghen is MIT.

### The JS API (1.33.0, from `node/index.d.ts`)

**The functions:**

- `transform`, `bundle` and `bundleAsync`, the last with a `resolver` that has `read` and `resolve`.
- `transformStyleAttribute`, `browserslistToTargets` and `composeVisitors`.

**The options:**

- `minify`, `sourceMap` and `inputSourceMap`.
- `targets`, plus `include` and `exclude` bit flags of `Features`, such as `Features.LightDark`.
- `drafts.customMedia`, `nonStandard`, `cssModules`, `analyzeDependencies` and `errorRecovery`.
- `visitor`, and `customAtRules`.

**The result:** `{ code, map, warnings, dependencies, exports }`.

**Visitors** are named callbacks that Lightning CSS calls while it walks the parsed stylesheet. A callback may return a replacement. The ones available:

- `StyleSheet` and `StyleSheetExit`, which receive the whole tree.
- `Rule` and `RuleExit`, either as one function or keyed by rule type. The keys include `unknown` (by at-rule name) and `custom`.
- `Declaration` and `DeclarationExit`.
- `Function` and `FunctionExit`, keyed by function name.
- `Token`, keyed by token type.
- `Variable` and `VariableExit`, which see `var()`.
- `EnvironmentVariable` and `EnvironmentVariableExit`, which see `env()`.
- Value visitors: `Color`, `Length`, `Url`, `Selector`, `MediaQuery` and the rest.
- Since 1.32.0, a visitor may also be a function that receives `addDependency` ([v1.32.0](https://github.com/parcel-bundler/lightningcss/releases/tag/v1.32.0)).

**What a visitor can return:**

- A function or token visitor may return tokens, typed values, or `{ raw: "…" }`, which Lightning CSS parses like CSS.
- A declaration may be returned typed, or as `{ property, raw }`.
- A media query may be returned as `{ raw }`.
- **A rule can only be returned as a typed tree.** There is no raw form for rules. The workaround proved in spikes 3 and 11 is to parse CSS text in a nested `transform` whose `StyleSheet` visitor captures `rules`, then return those.

**The order of calls,** from spikes 4 and 11:

- A declaration is entered, then its functions are visited, then it is exited.
- With one generic `Rule`/`RuleExit` function, a layer block is entered before the rules inside it and exited after them.
- The keyed `RuleExit` for `layer-block` did not fire in spike 4, though the generic one did. This may be a bug (**unconfirmed**).

**Custom at-rules** (`customAtRules`, since 1.19.0):

- **The prelude** (the part after the at-rule's name) is declared with a CSS syntax string. `<custom-ident>+` takes identifiers only. `*` takes any tokens as a list. `<dashed-ident>` is refused as "invalid syntax string" (spike 6).
- **The body** is `declaration-list`, `rule-list`, `style-block` or none.
- An at-rule that is not declared is an **unknown at-rule**. It is kept as raw tokens with a warning, "Unknown at rule: @x", and can be visited with `Rule.unknown`.
- Since 1.33.0, an unknown at-rule is printed where it stood, inside its rule ([#1233](https://github.com/parcel-bundler/lightningcss/pull/1233)). On 1.32.0, with nesting lowered, `@apply` was hoisted out of its rule (spike 12c).

**Warnings and errors:**

- `result.warnings` holds `{ message, type, loc: { filename, line, column } }`.
- **A visitor has no way to add a warning.**
- A syntax error throws with `loc` and `fileName`.
- **An error thrown inside a visitor comes out as a plain `Error` with no location** (spike 8).
- Rules carry a `loc`; declarations and functions carry none.
- `errorRecovery: true` drops an invalid rule or declaration with a warning. An unknown at-rule is kept either way.

**Source maps:**

- `sourceMap: true` writes a map, and `inputSourceMap` continues one from an earlier step.
- Rules parsed from generated text keep the line of that text and are counted against source 0, the input file. An adapter would have to rewrite their `loc` to the directive's own (spike 8).

**Comments are dropped,** even unminified, except a `/*! … */` licence comment at the top (spike 7; [#43](https://github.com/parcel-bundler/lightningcss/issues/43) and [#366](https://github.com/parcel-bundler/lightningcss/issues/366), both open).

**The printer rewrites what it keeps, minified or not:**

- `#ffffff` becomes `#fff`.
- `0.875rem` becomes `.875rem`.
- `'dark'` becomes `"dark"`.
- `color-mix()` of literal colours is computed to a colour, even with no targets.
- A layer order statement is folded into the layer blocks that follow it. **The blocks are moved into the declared order** (spike 4). The cascade comes out the same.

### Lowering: what it writes for older browsers

All of this was observed in spike 7, at Safari 15 / Chrome 100 targets and at Vite 8's defaults.

| CSS | What Lightning CSS writes for older targets |
|---|---|
| Nesting | Flattened (`.a:where(:hover)`) |
| `oklch()`, `lab()` | An sRGB fallback, then the colour again, or an `@supports (color: lab(0% 0 0))` block inside a custom property's layer |
| `color-mix()` | Computed when its arguments are literal; left alone around a `var()` |
| `light-dark()` | `var(--lightningcss-light, A) var(--lightningcss-dark, B)`. The two flags are declared only where a `color-scheme` declaration sits in the same sheet. Can be turned off with `exclude: Features.LightDark` |
| `@custom-media` | Expanded and removed under `drafts.customMedia` when targets are given; kept as written with no targets |
| `@custom-selector` | Not supported: kept with an "Unknown at rule" warning, so `postcss-custom-selectors` is still needed |
| `@property` | Kept as written; not polyfilled |
| `@layer` | Kept as written at every target; not polyfilled |
| Unknown at-rules (`@mixin`, `@apply`, `@toghen`) | Kept in place as raw tokens. Their contents are not lowered or minified |

### Mixins and custom functions, in the standard and in browsers

The spec is CSS Mixins and Functions Level 1:

- **Editor's Draft of 4 November 2026** ([css-mixins-1](https://drafts.csswg.org/css-mixins-1/)). It defines `@function`, `@mixin`, `@contents`, `@apply` and a new `@private`. It warns that the `@mixin` feature is experimental and "much less stable than @function".
- **The only W3C publication** is the First Public Working Draft of 15 May 2025, which covers functions only.

Browser support:

- **`@function`**:
  - Chrome has shipped it since Chrome 139, 5 August 2025 ([chromestatus](https://chromestatus.com/feature/5179721933651968)).
  - The Safari 27.3 beta notes list it ([notes](https://developer.apple.com/documentation/safari-release-notes/safari-27_3-release-notes)). The beta's date is **unconfirmed**.
  - Firefox does not have it ([bug 1950366](https://bugzilla.mozilla.org/show_bug.cgi?id=1950366)).
- **`@mixin`, `@apply` and `@contents`**:
  - Chromium has them behind a flag only. BCD lists Chrome 141 behind experimental web platform features ([BCD](https://github.com/mdn/browser-compat-data/blob/main/css/at-rules/mixin.json)), and chromestatus says "Proposed" ([entry](https://chromestatus.com/feature/5108022310469632)). `postcss/specs.md:395` says the same of Chromium 153.
  - WebKit and Gecko have nothing. Their standards positions are open.

Lightning CSS and mixins:

- It has no native mixins and no `@function`. Issue [#788](https://github.com/parcel-bundler/lightningcss/issues/788) is open with no reply from the maintainers. An outside pull request for `@function` only, [#1356](https://github.com/parcel-bundler/lightningcss/pull/1356), is unmerged.
- Its docs show a mixin visitor built on `customAtRules`. It runs on 1.33.0 for the docs' own example, and fails on Toghen's shape (see the next section).
- The CSS mixins polyfill Toghen uses, `@csstools/postcss-mixins`, is a PostCSS plugin. Its latest release is 1.2.0 (17 November 2026); Toghen pins 1.1.0.

### The visitor regression

Since 1.30.2, a visitor that returns a node holding a `null` optional field fails with "failed to deserialize".

- It is tracked in [#1065](https://github.com/parcel-bundler/lightningcss/issues/1065). The fix, [#1357](https://github.com/parcel-bundler/lightningcss/pull/1357), is unmerged.
- Devon Govett wrote on 22 November 2026 that it still happens on 1.33.0 ([#1081](https://github.com/parcel-bundler/lightningcss/issues/1081)).
- **Reproduced here, on 1.32.0 and 1.33.0** (spikes 6 and 10):
  - A `StyleSheet` visitor that returns its own argument unchanged fails on `.a{color:var(--x)}` with "expected an object-like struct named Specifier, found ()". So does any returned rule holding a `var()`, which is every rule Toghen writes.
  - The workaround, deleting every `null` member before returning, works. It is not exact: a percentage came back `1.0%` instead of `1%` on ui's stylesheet, which is the same value printed differently.

### PostCSS interop

- **There is no bridge that runs PostCSS plugins inside Lightning CSS.** The community package [`postcss-lightningcss`](https://www.npmjs.com/package/postcss-lightningcss) (1.1.2, 25 October 2026) works the other way round: it runs Lightning CSS as a PostCSS plugin.
- Lightning CSS's docs say "you may still need PostCSS for more custom plugins like TailwindCSS" ([docs](https://lightningcss.dev/docs.html)).
- On visitor speed, the 1.18.0 release notes say JS visitors are "around 2x slower" than Lightning CSS without them, and "around 6x faster than equivalent postcss plugins" ([v1.18.0](https://github.com/parcel-bundler/lightningcss/releases/tag/v1.18.0)).

### The runtime: native binaries and WASM

- **Native:** `lightningcss` loads one native binary per platform from an optional dependency: eleven platforms, including Linux musl and both Windows ABIs. On macOS arm64 the binary is 8.2 MB. It needs `detect-libc`.
- **WASM:** `lightningcss-wasm` is 16 MB. Since 1.18.0 it runs visitors, and since 1.21.8 `bundle` ([#373](https://github.com/parcel-bundler/lightningcss/pull/373), [#583](https://github.com/parcel-bundler/lightningcss/pull/583)). Its docs page still says the opposite. Spike 10 ran a `t()` visitor through it.
- **Measured speed** of minifying ui's 92 KB stylesheet: native 1.4 ms, WASM 8.1 ms. For comparison, PostCSS is 332 KB with no native code.

### Prior art

Every major adopter runs its own transforms first and uses Lightning CSS for the standard part afterwards: lowering, prefixing, minifying.

**Tailwind CSS v4** (`@tailwindcss/node` 4.3.5, 6 November 2026, pins `lightningcss` 1.33.0 exactly):

- Its directives (`@theme`, `@utility`, `@apply`) go through Tailwind's own parser before Lightning CSS sees anything.
- Lightning CSS then "optimizes", only in production builds or with `--minify`.
- Workarounds Tailwind needed ([optimize.ts](https://github.com/tailwindlabs/tailwindcss/blob/main/packages/%40tailwindcss-node/src/optimize.ts)):
  - It excludes `LightDark`, `LogicalProperties` and `DirSelector` lowering.
  - It runs Lightning CSS twice, so that rules are merged after nesting is flattened.
  - It patches the printed `@media not (`.
  - It filters warnings for selectors Lightning CSS does not know.
  - It patches the native-binary loader for its standalone build ([patch](https://github.com/tailwindlabs/tailwindcss/blob/main/patches/lightningcss@1.33.0.patch)).
- Its pass-through of native `@apply --mixin` notes that "Lightning CSS does not yet support this syntax" ([#19427](https://github.com/tailwindlabs/tailwindcss/pull/19427)).

**Vite 8** (8.3.8 checked in its published code):

- `build.cssMinify` defaults to Lightning CSS. It is called with `targets` from `build.cssTarget`, whose default `baseline-widely-available` is Chrome 111, Edge 111, Firefox 114 and Safari/iOS 16.4.
- It spreads `css.lightningcss` into that call, so `exclude` can be set there.
- `css.transformer: 'lightningcss'`, which replaces PostCSS altogether, is still marked experimental and still defaults to `'postcss'` ([options](https://vite.dev/config/shared-options)).

**Parcel 2:**

- Lightning CSS has been its default CSS transformer since 2.4.0, 22 March 2022.
- The default pipeline runs PostCSS first, then Lightning CSS ([default config](https://github.com/parcel-bundler/parcel/blob/v2/packages/configs/default/index.json)).

**Bun:**

- Its CSS parser and bundler is "a direct port of LightningCSS" to Zig, since v1.1.30 of 8 October 2024 ([blog](https://bun.com/blog/bun-v1.1.30), [docs](https://bun.com/docs/bundler/css)).
- The version that dropped the experimental flag is **unconfirmed**.

**Next.js / Turbopack:**

- Turbopack uses Lightning CSS, still runs PostCSS configurations in a Node worker pool, and keeps 5 decimal digits where webpack kept 10 ([Turbopack](https://nextjs.org/docs/app/api-reference/turbopack)).
- `experimental.useLightningcss` is for webpack only.

**Rspack and Rsbuild:**

- `builtin:lightningcss-loader` "can replace the postcss-loader and autoprefixer" ([docs](https://rspack.rs/guide/features/builtin-lightningcss-loader)).
- The Lightning CSS minimizer is Rspack's default.

---

## Findings

Each finding was checked against the code, the docs or a spike, and says where.

**Toghen today**

- **F1. The plugin is one file of 2,199 lines, and most of it is host-free.** Its output nodes are plain objects (`{prop, value}`, `{selector, nodes}`, `{name, params, nodes}`, `{text}`), and PostCSS turns them into nodes. The PostCSS-specific parts are few:
  - the `Once` hook that writes the mixins before the polyfill reads the sheet;
  - walking `parent` to find the enclosing rule or layer;
  - `node.warn` and `node.error`;
  - `replaceWith`;
  - the `proxyOf` key for per-sheet state.

  The value spelling (`valueToCss`, colours, shadows, `font`, family quoting) and the directive logic do not depend on PostCSS. *Checked*: `postcss/src/index.ts:94-315, 501-519, 1139-1144, 2016-2024, 2148`. *Status*: Confirmed.
- **F2. Lib owns the shared names but not the value spelling.** Lib exports `cssPropertyName`, `cssIdentifier`, `cssLinkName`, `getContextSelector` and `getLightDark`. `valueToCss` and the rest of the value spelling live in postcss. Lib "writes nothing out: an adapter serialises", and lib says two adapters "cannot import from each other". *Checked*: `lib/src/index.ts:63-64`, `lib/specs.md:11, 944`, `lib/src/utils/css-name.ts:31`. *Status*: Confirmed.
- **F3. The components need the PostCSS mixins polyfill in every pipeline.** The plugin writes `@mixin`, `@contents` and `@apply` targets. The guide installs `@csstools/postcss-mixins`, lists it after Toghen, and adds a nesting polyfill only where needed. *Checked*: `postcss/specs.md:395`, `postcss/guide.md:12, 17`. *Status*: Confirmed.
- **F4. ui's build is one PostCSS pipeline:** Toghen (`references: 'var'`, `typography: 'font'`, `onUnresolved: 'throw'`), then the mixins polyfill, then a small comment stripper. A hand-written regex inliner prepares the input. The stylesheet ships nested, for Figma's Chromium. *Checked*: `ui/scripts/build.mjs:47-54, 83-88`. *Status*: Confirmed.
- **F5. ui's entry stylesheet is not valid CSS for a standard bundler.** `@toghen mixins toghen-plugin;` stands before the `@import`s, and CSS allows `@import` only before every other rule. Lightning CSS's bundler therefore ignored all nineteen imports (spike 12). It works today only because ui's inliner is a regex. *Checked*: `ui/src/index.css:15-17`. *Status*: Confirmed.
- **F6. The Figma plugin does not minify its CSS.** esbuild minifies the JavaScript only, in production. The design system's stylesheet is injected as text into `ui.html`, which `code.js` embeds as a string. *Checked*: `figma/build.mjs:15, 24, 36-40, 65-68`. *Status*: Confirmed.
- **F7. figma's tests read ui's built stylesheet by text.** They match `--tg-bg-selected: #0c8ce91a;` and four other lines. A minified `styles.css` (`--tg-bg-selected:#0c8ce91a`) would fail them, so any minifying belongs to figma's build output, not to ui's export. *Checked*: `figma/test/uiStyles.test.ts:28, 34-40`. *Status*: Confirmed.
- **F8. No doc names Lightning CSS, and one open item is about minifiers.** The specs leave unmeasured "whether a minifier keeps the empty value" of a chain flag (`--theme--dark: ;`). *Checked*: grep across all seven repositories, and `postcss/specs.md:376`. *Status*: Confirmed.
- **F9. The pipeline skill teaches the PostCSS build.** *Checked*: `skills/plugins/toghen/skills/pipeline/commands.md:96-115`. *Status*: Confirmed.

**Lightning CSS against what Toghen writes**

- **F10. Lightning CSS breaks token paths with a number after a dot.** In `t(color.gray.500)` the tokenizer reads `.500` as the number 0.5. A visitor is handed `color`, `.`, `gray`, then 0.5, and `500` cannot be recovered. `.100` arrives as 0.10000000149011612. `elevation.raised.0.color`, the member path the reference documents, arrives with a bare 0.

  Printed back, `t(color.gray.500)` becomes `t(color.gray.5)` and `t(tg.sp.0)` becomes `t(tg.sp0)`. ui's real build failed on `tg.sp0` when its sources went through Lightning CSS's bundler first. PostCSS keeps the text. *Checked*: spikes 12b and 13. *Status*: Confirmed.
- **F11. Lightning CSS lowers Toghen's `light-dark()` pairs into a broken form.** A Vite 8 production build does this by default, after the PostCSS plugin. At Vite 8's default targets, `:root{--color-bg:light-dark(#ffffff,#000000)}` becomes `:root{--color-bg:var(--lightningcss-light,#fff)var(--lightningcss-dark,#000)}`. The flags are declared only where `color-scheme` is declared in the same run.
  - **No `color-scheme` in that sheet,** which is the case the plugin leaves to the app: the value is two colours side by side, invalid wherever it is used.
  - **With one,** it is decided where `--color-bg` is declared, the root. A subtree set to `color-scheme: dark` inherits the light value, so the switching on a subtree that the guide promises is lost.

  *Checked*: spikes 7 and 14; Vite 8.3.8's `minifyCSS` and `ESBUILD_BASELINE_WIDELY_AVAILABLE_TARGET` in its published `dist/node/chunks/node.js`. The CSS rule that a `var()` is substituted where the property is declared is the one the specs already measured (`postcss/specs.md`, *Declaring a context*). *Status*: Confirmed in output; the effect in a browser is derived, **not measured**.
- **F12. What survives a Lightning CSS pass of Toghen's output:**
  - Chain flags with an empty value (`--theme--dark: ;`), minified, and the `initial` resets.
  - The custom properties. The checks in figma's tests still match on the unminified output.
  - The layer order statements, folded into blocks with the cascade order kept.
  - `@property` rules.
  - Unknown `@mixin` and `@apply`, in place (1.33.0).

  **Comments do not survive,** so `descriptions: 'comment'` output does not either. *Checked*: spikes 4, 5, 7 and 9. *Status*: Confirmed.
- **F13. Lightning CSS needs 1.33.0 or later wherever an unexpanded `@apply` reaches it with nesting lowered.** On 1.32.0, `.button{display:inline-flex;@apply --button;color:red}` at Safari 15 targets printed `@apply --button;` at the top level, outside its rule. 1.33.0 keeps it inside. The guide's `preserve: true` keeps `@apply` beside its expansion, so a later Lightning CSS step does see it. *Checked*: spike 12c. *Status*: Confirmed.
- **F14. A thin Lightning CSS layer over a text-writing core reproduces today's output** on the running example's tutorial stylesheet, after both are printed by Lightning CSS. In spike 11 the core was the existing plugin, run on a one-directive sheet. That layer is called a *shell* below. Its whole job:
  - Find each directive with `Rule.unknown` and each read with `Function.t` or `Function.toghen`.
  - Track the enclosing layers with a generic `Rule`/`RuleExit`.
  - Track the declaration's property with `Declaration`, for the rule that `--a: t(a)` must not read itself.
  - Parse the core's text with a nested `transform`, strip the `null`s, and return the result.

  Inside a rule, `@toghen declare` was written as a `nested-declarations` rule of raw declarations (spike 15). *Checked*: spikes 11 and 15. *Status*: Confirmed for that stylesheet, whose paths have no numeric segment.
- **F15. The host is about 1% of the build.** On ui's real stylesheet:
  - The whole build took 351 ms; Toghen alone 358 ms (this run's noise exceeds the polyfill's cost); loading and resolving the system 5 ms.
  - PostCSS parsing and printing ui's 92 KB output took 3.5 ms, Lightning CSS 1.4 ms.
  - A Lightning CSS `StyleSheet` visitor that only reads the tree took 6.6 ms; one that returns it, with the workaround, 47 ms.

  So a Lightning CSS host would not make Toghen's step faster, and a visitor that hands the whole tree to JS would make it slower. *Checked*: spikes 9 and 10. *Status*: Confirmed on this machine (Apple silicon, Node 25.6.1).
- **F16. Minifying ui's stylesheet saves about 15 KB, and esbuild nearly matches Lightning CSS.**

  | Build | Without fonts | With fonts |
  |---|---|---|
  | Today | 92.0 KB | 435.8 KB |
  | Lightning CSS, minified | 76.9 KB (77.9 KB at Safari 15 targets) | 420.6 KB |
  | esbuild 0.21.5, minified | 78.9 KB | 422.7 KB |

  esbuild is already figma's dependency. For scale, the built `ui.html` is 652 KB and `code.js` 1.67 MB. *Checked*: spikes 9 and 16; `figma/dist`. *Status*: Confirmed.
- **F17. Nothing in Toghen's own builds needs lowering.** ui ships nested CSS for Figma's Chromium on purpose, and uses no `light-dark()` (no occurrence in its built sheet). *Checked*: `ui/scripts/build.mjs:8-9`; grep of the built sheet. *Status*: Confirmed.
- **F18. A body-less custom at-rule is printed without its `;`.** With `customAtRules: { foo: {} }`, `@foo; .a{color:red}` printed `@foo.a{color:red}` minified, and `.b{@foo;color:blue}` printed `@foocolor:#00f`. An adapter that declares `@toghen` as a custom at-rule and leaves a failed directive in place, as the postcss rule requires, would corrupt the minified sheet. Read as an unknown at-rule, it prints correctly. *Checked*: spike 6. *Status*: Confirmed. Whether it is reported upstream is **unchecked**.

---

## What the spikes showed

The spikes are in `/private/tmp/claude-501/-Users-egildo--claude-spares-spare-Fpfl06/85778d0b-8740-48e9-8b29-268e30e7de4d/scratchpad/lightningcss/spikes/`, with Lightning CSS 1.33.0 and `lightningcss-wasm` 1.33.0. The 1.32.0 check ran in the sibling folder `lightningcss-132/`. Spikes 12c and 13 to 16, and the one-liners after spike 7, were run as inline commands and not saved; each is described well enough here to type again.

1. **`s1-function.mjs`: a `Function` visitor replaces `t(path)`** with `{ raw }`.
   - The name is matched whole: `inset(1px)` is untouched. Strings are untouched, which postcss's review fix F6 must still build.
   - A call the visitor leaves alone stays in the output, as the postcss rule wants.
   - `url(t(x).png)` is a bad URL, and the declaration is dropped.
2. **`s2-atrules.mjs`: `@toghen` directives.** Left alone, they are kept in place (in a layer, in a rule), each with an "Unknown at rule" warning. `Rule.unknown.toghen` receives the tokens after the name and a location. `customAtRules` with prelude `*` receives them as a token list and silences the warning; with `<custom-ident>+` it refuses `declare ds.space, …`.
3. **`s3-emit.mjs`: text parsed in a nested `transform` and returned from the visitor works,** with layers, `@media` and `@property`. The layer order statement was folded into the blocks.
4. **`s4-layers.mjs`: order statements are folded and blocks reordered,** minified or not. The keyed `RuleExit` for `layer-block` did not fire; the generic one did.
5. **`s5-mixins.mjs`: Toghen's real `@mixin`/`@apply` output survives in place as raw tokens.** It is not lowered or minified inside, and comes with two warnings.
6. **`s6`/`s6b-mixin-visitor.mjs`: the docs' mixin visitor.**
   - It runs on the docs' own example.
   - On Toghen's shape (`var()` inside) it fails with #1065, and works with `null`s stripped.
   - `<dashed-ident>` is not a valid prelude syntax.
   - A body-less custom at-rule is printed without `;` (F18).
7. **`s7-lowering.mjs` and the following one-liners:**
   - `light-dark()`, `oklch()` and `color-mix()` are lowered as in the table above.
   - `@custom-media` is expanded; `@custom-selector` is not.
   - `@property` and `@layer` are kept.
   - Comments are dropped, but not a leading `/*!`.
   - `--x: ;` survives minifying.
8. **`s8-errors-maps.mjs`:** errors and maps behave as described above.
   - A visitor's thrown error carries no location.
   - A syntax error has `loc`.
   - Declarations have no `loc`.
   - Generated rules keep the line of the text they came from.
9. **`s9-ui.mjs`: ui's real build and Lightning CSS on its output.** The timings and sizes are in F15 and F16. ui's build functions write nothing; the outputs went to `out/`.
10. **`s10-split.mjs`:**
    - The timings split: Toghen's own work against the host's.
    - The WASM build minifies in 8.1 ms and runs a `t()` visitor.
    - An identity `StyleSheet` visitor fails with #1065 on ui's sheet; with the workaround it takes 47 ms and prints `1%` as `1.0%`.
11. **`s11-shell.mjs`: the shell of F14,** around the existing plugin used as a text-writing core. The output is identical to today's after Lightning CSS normalises both. A read keeps the rule that `--a: t(a)` does not read itself.
12. **`s12-ui-bundle.mjs`, `s12b-ui-bundle.mjs` and the 1.32.0 check (12c):**
    - Lightning CSS's bundler ignores ui's imports because of F5.
    - With `@toghen mixins` moved after them in memory, it inlines them and keeps `@toghen` and `@apply` in place, in 2.7 ms.
    - The bundled sheet then fails in Toghen, on `tg.sp0` (F10).
    - 1.32.0 hoists `@apply` out of its rule when lowering nesting; 1.33.0 does not (F13).
13. **The numeric-path check:** what a visitor receives for `t(color.gray.500)`, `t(space.100)`, `t(tg.sp.0)` and `t(elevation.raised.0.color)`, and what Lightning CSS prints for them against PostCSS.
14. **`light-dark()` at Vite 8's targets:** the lowered output of F11, with and without `color-scheme` in the sheet.
15. **`@toghen declare` inside a rule** written as `nested-declarations` of raw declarations: it prints as plain declarations of the rule, minified or not, and is split into a sibling rule when nesting is lowered.
16. **esbuild's CSS minifier on ui's sheet,** with figma's installed esbuild 0.21.5: 17 ms, 78.9 KB.

---

## Every feature, mapped

Each `@toghen/postcss` feature against a Lightning CSS mechanism, or a gap.

| Feature | Today in postcss | In Lightning CSS | Gap or risk |
|---|---|---|---|
| `t(path)` / `toghen(path)` in a value | A regex over the declaration's text, with a name boundary (`index.ts:429`) | `Function: { t, toghen }`, returning `{ raw }`. The name is matched whole by the parser (spike 1) | **A path with a number after a dot cannot be read** (F10): `gray.500`, `space.100`, `raised.0.color`. A quoted path, or a pre-scan of the source text, would be needed |
| `as value`, a mode after the comma, a member path, leaving out the collection | Parsed from text | The same, from tokens put back into text | Only as above: numeric segments, and modes holding numbers |
| `--a: t(a)` not reading itself | Reads `decl.prop` | A `Declaration` visitor before the function records the property (spike 11) | None |
| `t(` inside strings and `url()` | Replaced today; a fix is in the review round (F6) | Left alone by construction (spike 1) | None |
| `t()` in an at-rule's parameters | Ignored today; the review round reports it (F7) | `@media (min-width: t(x))` is a syntax error without `errorRecovery` | Behaves differently; either way it is reported |
| `@toghen contexts`: defaults and context blocks | Plain nodes replacing the at-rule | `Rule.unknown.toghen`, then text parsed to a tree (spikes 3, 11) | Needs the nested-parse workaround and the `null` workaround (#1065) |
| The same, inside a named layer, with its order statement | Walks `parent` (`index.ts:2116`), and remembers one order per layer per sheet | Layers tracked with the generic `Rule`/`RuleExit` (spike 3) | **Lightning CSS folds the statement and reorders the blocks** (spike 4). The cascade is the same, but the "one order per layer" refusal names a statement that is no longer in the output |
| `light-dark()` pairs | Written as `light-dark(a, b)`; the app sets `color-scheme` | Written the same | **Lowered in a broken form by Lightning CSS at older targets** (F11). Needs `exclude: Features.LightDark` |
| `registerProperties`: `@property` | Plain at-rule nodes | `PropertyRule`, or parsed text; kept as written | None. Not polyfilled anywhere |
| `bind: 'chain'`: links, flags, `--x: ;` | Plain declarations | Raw declarations; the empty value survives minifying (spike 7) | None. This also answers the open minifier item for Lightning CSS (F8) |
| `@toghen mixins`: `@mixin`, `@contents`, part rules, `--when-` mixins | Written in `Once`, so the polyfill listed after finds them | Text parsed into unknown-rule trees; kept in place | **Nothing expands `@apply` without PostCSS** (F3). A visitor-based expander is possible but meets #1065 and F18, and it would be Toghen writing a polyfill of a draft |
| Two paths, one mixin name: always an error | `atRule.error` | A thrown `Error` | **No location** (spike 8) |
| `@toghen spellings` | Writes `@custom-selector` and `@custom-media` | `@custom-media` is expanded under `drafts.customMedia`; `@custom-selector` is unknown | `postcss-custom-selectors` is still needed |
| `@toghen declare` inside a rule | Declarations replace the at-rule | A `nested-declarations` rule of `{ property, raw }` (spike 15) | Split into a sibling rule when nesting is lowered. Same cascade |
| `@t` short forms | One handler for both names | `Rule.unknown.t` beside `toghen` | None |
| Retired forms (`:toghen()` selectors, old at-rules, old options) | `Rule` visitor, at-rule sentences, `Once` | A `Selector` visitor or a style-rule visitor; at-rules as above; options checked when the plugin is called | None. The messages need a channel (next row) |
| `onUnresolved: 'warn'` | `node.warn(result, …)`, with the node's position | **No API for a visitor to warn** | The adapter needs its own channel: a callback, or a wrapper around `transform` that merges its messages into `warnings`. Under Vite or Parcel they would not appear in the tool's own warning list |
| `onUnresolved: 'throw'` | `node.error()` → `CssSyntaxError` with line and column | A thrown `Error` with no location | The adapter adds the enclosing rule's `loc`. A declaration's own line is not available |
| `onUnresolved: 'ignore'` | Leaves the call | Return nothing | None |
| A failed directive left in the output | The at-rule stays | Stays as an unknown at-rule | **Must not be declared in `customAtRules`** (F18) |
| Source positions of generated nodes (review fix F8) | `source` copied from the directive | Each generated rule's `loc` rewritten to the directive's | The adapter's own work |
| `descriptions: 'comment'` | Comment nodes | **No comment node; comments dropped** | Not possible. The option would be refused in a Lightning CSS adapter, and descriptions are lost wherever Lightning CSS runs after PostCSS |
| Value spelling (`valueToCss`, colours, shadows, `font`, family quoting, cubic-bezier) | Pure functions in the adapter | The same functions | None, once shared. Lightning CSS's printer re-spells colours and numbers anyway |
| mdn-data longhands | JSON through `require` | The same | None |
| The options `references`, `propertyName`, `typography`, `composites`, `rootSelector`, `defaultMode`, `origins`, `bind` | Plain options | The same | None |
| The `[object Object]` and `toghen(` leak checks | Text checks on the built plugin | Unchanged | None |
| Golden files, byte for byte | PostCSS's printer, with set spacing (`index.ts:1165-1166`) | Lightning CSS's printer re-spells everything | Goldens for a second adapter compare after both are printed by Lightning CSS (spike 11's method) |
| The pipeline | One `postcss([...])`, Toghen first | A visitor composed into the tool's own Lightning CSS call (Vite's `css.lightningcss.visitor`, Parcel's config) | **Lightning CSS must never see Toghen's input before Toghen does** (F10). A Lightning CSS-first bundling step breaks paths |
| Peer dependencies | `postcss ^8`, `@toghen/lib` | `lightningcss >=1.33.0` (F13), `@toghen/lib` | Native binary per platform. MPL-2.0 licence |

---

## The options

### (a) A second adapter, `@toghen/lightningcss`, beside postcss

**What it is.** A Lightning CSS visitor that consumers compose into their Lightning CSS step: Vite's `css.lightningcss.visitor` under `css.transformer: 'lightningcss'`, Parcel's or Rspack's configuration, or a direct `transform()`.

It must not copy the 2,199 lines, because two copies of the spelling drift. So it needs option (d) first: a core that both adapters share. Lib's own rule is that adapters cannot import from each other (F2).

**What it costs.**

- The core extraction from (d).
- A shell of a few hundred lines. Spike 11's was 40 lines, borrowing the whole plugin.
- Workarounds for #1065 and F18 until upstream fixes them.
- A warning channel, location rewriting, and a way to read numeric paths (F10, a question below).
- A new repository on Codeberg, with house's shared files, its own `CLAUDE.md` source in house, and a row in house's README.
- A constitution amendment for the new package's rules (minor version).
- A second release train, and the pipeline skill re-pinned.

**What breaks for consumers of `@toghen/postcss`.** Nothing. They keep the PostCSS adapter.

**For a consumer who uses only Lightning CSS:**

- Tokens, contexts, reads, `declare` and `@custom-media` work.
- `descriptions: 'comment'` does not.
- The components do not work: their `@mixin` definitions pass through, and no tool and no shipping browser expands `@apply`.

**What the tests become.**

- The 375 tests run against both shells through one harness. Each test is written once and asserts on CSS normalised through Lightning CSS's printer.
- The PostCSS shell keeps its byte-for-byte goldens.
- The Lightning CSS shell gets the normalised comparison of spike 11, plus tests of its own: numeric paths, warnings, `loc`, and the #1065 workaround.

**What the docs become.**

- The new repository gets a guide, reference, specs and decisions.
- postcss's specs move "what CSS a directive writes" to the core's docs, and keep "how PostCSS runs it" (`Once`, the polyfill's order, `result`).
- The tutorial stays on PostCSS, because the components need the polyfill.

**Which repositories change.**

- postcss: the core extraction.
- The new repository.
- house: the README, the `claude-md` source, the workspace's `pnpm-workspace.yaml`, the constitution.
- skills: the pipeline skill.
- ui and figma: no change.

**The mixins question.** The adapter writes `@mixin` as today. Expansion is the consumer's tools' or the browser's. Today that means: none without PostCSS.

### (b) Replace `@toghen/postcss` with a Lightning CSS adapter

**What it is.** As (a), with the PostCSS adapter withdrawn.

**What it costs.** Everything in (a), plus a deprecation path.

**What breaks for consumers.** Every consumer's configuration changes from a PostCSS plugin to a Lightning CSS visitor. Three things are lost:

- `descriptions: 'comment'`;
- warnings in PostCSS's `result`, with exact positions;
- the one pipeline with the mixins polyfill.

The components would need a PostCSS pass anyway, for the polyfill. Consumers would then run two tools where they run one.

**The tests and docs.** The goldens are rewritten in Lightning CSS's spelling. The tutorial and every guide example change.

**Which repositories change.** postcss is retired or renamed; ui's build changes; skills change; house changes.

**The mixins question.** Not answered. This is why (b) is not viable today.

### (c) Keep the adapter on PostCSS, and switch the builds to Lightning CSS

**What it is.** Put Lightning CSS after PostCSS in ui's build, or in figma's build on the injected stylesheet, to lower and minify.

**What it costs.** A dependency: 8 MB native per platform, MPL-2.0. A few lines per build.

**What it gives.** About 15 KB (F16). There is nothing to lower (F17). esbuild, already figma's dependency, does the minifying to within 2 KB.

**What breaks.**

- Minifying ui's exported `styles.css` would fail figma's tests (F7). Minifying in figma's build would not.
- Lightning CSS must run after Toghen, never before. ui's inliner cannot become Lightning CSS's bundler without moving `@toghen mixins` after the imports (F5) and solving F10.

**The tests and docs.** A line in ui's or figma's specs. figma's leak check still applies.

**The mixins question.** Untouched: PostCSS still expands them.

### (d) A core that is not tied to any host, with thin PostCSS and Lightning CSS shells

**What it is.** Split `postcss/src/index.ts` into two parts:

- **The core:** given the `DesignSystem`, the options, a directive and where it sits (its enclosing layers, whether it is in a rule, the declaration's property), it writes CSS text, or plain nodes, and reports messages through a callback that carries a location the host supplies.
- **A PostCSS shell:** `Once`, the visitors, `result.warn`, `node.error`, the node replacement.

Spike 11 proved the shape with text as the core's output. Plain nodes would also do: they are already what the plugin builds.

**What it costs.** A refactor with no change of behaviour. The goldens stay byte for byte and the 375 tests stay green.

It touches the very lines the review round's postcss section changes:

- positions (F5),
- `source` on generated nodes (F8),
- strings and `url()` (F6),
- layer names through `ident()` (F3).

So it starts after that round merges.

**What breaks.** Nothing.

**The tests.** Unchanged, plus a few that call the core with no host.

**The docs.** postcss's specs gain a short section on the split.

**Which repositories change.** postcss only, until a second shell exists.

**The mixins question.** Untouched.

**The variant that moves the spelling into lib instead.** `valueToCss`, the colour, shadow and `font` spelling, and family quoting would go into lib. That needs the constitution amended: lib "writes nothing out" would change, and the nine questions would widen with value spelling under *css*. The Figma adapter does not need CSS value spelling, so by lib's own rule ("a question enters the core when two consumers ask it") only a second CSS adapter would justify it. Keeping the core inside the CSS adapter's own repository serves the same purpose without the amendment.

### (e) Lightning CSS as an environment Toghen's output must survive

**What it is.** The adapter stays on PostCSS, and the builds stay as they are. Toghen proves and documents what happens when Lightning CSS runs after it. That already happens by default in Vite 8's production builds, and in Parcel, Turbopack and Tailwind's optimise step. This option comes out of F10 to F13: Lightning CSS is already in consumers' pipelines, and it touches Toghen's output there.

**What it costs.**

- A test in postcss with `lightningcss` pinned at 1.33.0 as a dev dependency.
- A guide section, a specs line closing the minifier item, and a browser measurement for `light-dark()`.

**What breaks.** Nothing. It prevents breakage that exists today.

**The tests.** Every golden output goes through Lightning CSS at three settings:

- with no targets;
- at Vite 8's default targets, minified;
- the same with `exclude: Features.LightDark`.

Each run is checked for:

- no warnings beyond the known unknown at-rules;
- every custom property still declared;
- the layer order unchanged;
- the empty chain flags kept;
- `light-dark(` kept when excluded and rewritten when not, which pins the hazard so a Lightning CSS release that changes it is noticed.

**The docs.** A guide section, *When Lightning CSS runs after the plugin*:

- List Toghen before any plugin that runs Lightning CSS, Tailwind's optimise step included (F10).
- Exclude `light-dark()` lowering (`css.lightningcss.exclude` in Vite).
- Use 1.33.0 or later when `@apply` reaches it.
- Comments, and so descriptions, are dropped.

A specs line closes "Not measured: whether a minifier keeps the empty value" for Lightning CSS 1.33.0. A decisions entry records the spike that proved a second adapter's path.

**Which repositories change.** postcss: test, guide, specs, measurements. skills: one line in the pipeline skill, when it is next re-pinned.

**The mixins question.** Untouched, and stated: with Lightning CSS after PostCSS, the polyfill has already expanded them.

---

## Recommendation

**Choose (e) now.** It answers what Lightning CSS already does to Toghen's output in consumers' builds: `light-dark()` lowered into a broken form, and paths corrupted when Lightning CSS runs first. It needs no new package and changes no output.

**Leave Toghen's own builds as they are.** There is nothing to lower, and minifying saves about 15 KB of a 652 KB page.

**Keep (a), built on (d), as the known path for later.** Record in decisions that spike 11 proved it. Start it when four things hold:

- a consumer needs a pipeline without PostCSS (the loop's "real usage surfaces a gap");
- #1065's fix is released;
- the path question and the mixins question below are answered;
- the review round's postcss section has merged.

**Reject (b) on today's facts.** No pipeline without PostCSS can expand the mixins, so replacing the adapter would leave components without a working form.

---

## Units for the recommended option

Each unit works in its own worktree under `/Volumes/dev/Codeberg/Toghen-worktrees/`, on a branch cut from the repository's `main`, with the repository's `pnpm check` as its gate. Units A and B run in parallel. C follows B, because its wording depends on B's measurement.

| Unit | Repository | Worktree folder | Branch | Starts |
|---|---|---|---|---|
| A | postcss: the parity test | `postcss-lightningcss` | `lightningcss-parity` | After the review round's postcss section merges, so its fixes are in the goldens |
| B | postcss: the browser measurement | none (scratch, per `measurements/README.md`) | the fixture lands with C | Now |
| C | postcss: the docs | the same worktree as A | `lightningcss-parity` | After B |
| D | skills: the pipeline skill's line | `skills` | `lightningcss-note` | After postcss's release |

**A. The parity test.**

- Add `lightningcss` 1.33.0, pinned exactly, as a dev dependency.
- Add `test/lightningcss.test.ts`. It reads every `test/example-output/*.out.css` plus the tutorial's polyfilled outputs, runs Lightning CSS at the three settings of option (e), and asserts what option (e) lists.
- Add one test that pins F10: Lightning CSS's printing of `t(a.b.500)` is not the path. It is a guard on the guide's order rule, not a feature.
- Write it like the tests around it: synthetic tokens, the example's structured shapes.

**B. The browser measurement.**

- Add a fixture to `postcss/measurements/`, the next number after `c8`, holding a `light-dark()` modifier's output in three forms:
  - native;
  - lowered by Lightning CSS at Vite 8's targets with `color-scheme` on the root and a subtree set to dark;
  - lowered with no `color-scheme` in the sheet.
- Read the computed colours in Chromium and WebKit, through Playwright from a scratch folder as the README says.
- Record the result in `RESULTS.md`. It confirms or refutes the browser half of F11 before the guide states it.

**C. The docs.**

- In `guide.md`, after *Letting the app choose the scheme*, write the section described in option (e), with Vite's `css.lightningcss.exclude: Features.LightDark` as the example, and add a line to *Troubleshoot*.
- In `specs.md:376`, close the minifier item for Lightning CSS 1.33.0, as measured, and note that comments do not survive it.
- Put what the build found in the report; the integration writes the decisions entry.

**D. skills.** When the pipeline skill is next re-pinned to a postcss release that has C, add the order rule to `commands.md`'s CSS build. Follow the skills rule: read the release notes at the tag, update the pins and the date, and bump the plugin's version.

**Integration.** Review A to C against this plan, run `pnpm check` in postcss, rebuild the plugin in the local workspace and confirm no leak, write the dated decisions entry, and report. Merging, a version bump and a publish follow Egildo's word.

## Units for the deferred path, if the first question is answered (a) or (b)

These are listed so the cost is visible. They do not start before their gates.

| Unit | Repository | What | Gate |
|---|---|---|---|
| E | postcss | Extract the core (option d). No change of behaviour; goldens byte for byte | The review round's postcss section merged |
| F | postcss | The shared test harness: each test runs against a list of shells, with normalised comparison for any but PostCSS | E |
| G | new repository | The Lightning CSS shell, in its own worktree outside the workspace:<br>- `Rule.unknown` for directives, so a failed one prints correctly (F18), not `customAtRules`;<br>- `Function` and `Declaration` for reads;<br>- generic `Rule`/`RuleExit` for layers;<br>- text parsed back with `loc` rewritten;<br>- `null`s stripped until #1065 is released;<br>- messages through the chosen channel;<br>- `descriptions: 'comment'` refused with a sentence | E, and the answers on paths, mixins and the gate |
| H | postcss and the new repository | The path form chosen for numeric segments, in the core, so both shells read it | The path answer |
| I | house | The README table, the `claude-md` source, the workspace `pnpm-workspace.yaml`, the shared files copied, the constitution amended with the new package's rules | The first answer |
| J | skills | The pipeline skill: a Lightning CSS build beside the PostCSS one, re-pinned | G released |

E and I can run in parallel. F and H follow E. G follows F. J comes last.

---

## Risks

1. **Consumers' Vite 8 builds lower `light-dark()` today** (F11). It is outside the switch, and it is the most urgent thing this research found. Unit B measures it and unit C documents it.
2. **Lightning CSS before Toghen corrupts paths** (F10). Some corruptions are caught as "not found". Others may resolve silently to another token: `gray.500` becomes `gray.5`, which a palette may hold. Only the order rule prevents it.
3. **The visitor regression #1065** blocks any shell that returns rules holding a `var()`. The workaround is not exact (`1%` became `1.0%`). The fix is unmerged, and no release has come for four months.
4. **A body-less custom at-rule loses its `;`** (F18). A shell must not declare `@toghen` as a custom at-rule.
5. **No warning channel, and positions only per rule,** in a visitor. Messages from a Lightning CSS adapter would not appear where users of Vite or Parcel look.
6. **Comments are dropped.** `descriptions: 'comment'` cannot work under Lightning CSS.
7. **The mixins draft is unstable.** Its editors call `@mixin` "much less stable than @function", and no browser ships it. An expander written now would chase the draft.
8. **The runtime.** A native binary per platform: 8.2 MB on macOS arm64, eleven platforms, `detect-libc` on Linux. The WASM fallback is 16 MB and about six times slower. Tailwind had to patch the loader for its standalone build. The licence is MPL-2.0.
9. **Maintenance.** One main maintainer, 338 open issues, the crate pre-1.0, and a four-month gap since the last release.
10. **Lightning CSS's printer re-spells output.** A text check placed after it breaks: figma's tests, and any golden file.
11. **The review round's postcss section** touches what (d) would move: positions, sources, strings, layer names. Starting (d) before it merges means two rounds editing the same lines.

---

## Measurement plan

What must be measured before building, and how.

1. **Output parity on the running example.**
   - Under (e): unit A's test, every golden through Lightning CSS at three settings.
   - Under (a): spike 11's normalised comparison, extended to every golden and every test's output, with a list of each difference and why.
2. **Parity on ui's real stylesheet.** Needed only if the builds question is answered (b) or (c).
   - Compare today's sheet with the Lightning CSS-minified one, fonts aside: the set of declared custom properties, rules per layer, the layer order.
   - Then compute the styles of ui's gallery page (`ui/dev/gallery.html`) in Chromium and WebKit with each sheet, for every component and state, through Playwright in a scratch folder.
3. **`light-dark()` lowered, in a browser.** Unit B. This one is needed whatever the answers.
4. **Build time.** Already measured on this machine (F15):
   - ui's build: 351 ms, about 350 ms of it Toghen's own work.
   - Host parsing and printing: 3.5 ms for PostCSS, 1.4 ms for Lightning CSS.
   - A whole-tree visitor round trip: 47 ms.

   Repeat only under (a), with the shell on ui's sheet, reporting the host's share and the visitor's cost.
5. **The plugin bundle.** Already measured (F16). Re-measure on a fresh production build only if the builds question is answered (b) or (c): `dist/ui.html` and `dist/code.js` before and after, with and without fonts.
6. **Before unit G starts:**
   - #1065's state;
   - whether F18 is reported upstream;
   - whether the keyed `RuleExit` not firing is a bug;
   - a list of the numeric-segment paths in lib's example and ui's tokens, to size F10.

---

## Questions

1. **What should "the switch" mean?**
   - *Context*: Lightning CSS already runs after the PostCSS plugin in consumers' builds: by default in Vite 8, and in Parcel, Turbopack and Tailwind. There it lowers `light-dark()` into a broken form. Nothing in a pipeline without PostCSS can expand Toghen's mixins. A Lightning CSS host would not make Toghen faster: the host is about 1% of the build. A second adapter is feasible, and the spike proved its shape, but it needs a shared core, workarounds for an open regression, and answers on paths and mixins.
   - a. **Lightning CSS as an environment Toghen's output must survive**: a parity test, a guide section and a browser measurement. The adapter stays on PostCSS. Costs four small units and adds no package. *Recommended*: it fixes the exposure that exists today, and defers a second adapter until a consumer needs one, as the loop asks.
   - b. **A second adapter, `@toghen/lightningcss`, on a core extracted from the PostCSS plugin**: tokens, contexts, reads and `declare` for pipelines without PostCSS. Costs the extraction, a new repository and package, a constitution amendment, and a second release train. Components still need PostCSS.
   - c. **Replace `@toghen/postcss`**: one adapter, on Lightning CSS. Costs all of b, breaks every consumer's configuration, loses descriptions and exact warning positions, and still needs PostCSS for the components.

2. **Should Toghen's own builds use Lightning CSS?**
   - *Context*: ui ships nested CSS for Figma's Chromium, so there is nothing to lower. Minifying the stylesheet saves about 15 KB of 436 KB, most of which is embedded fonts. esbuild, already figma's dependency, comes within 2 KB of Lightning CSS. figma's tests read ui's exported stylesheet by text, so minifying ui's export would break them.
   - a. **Leave both builds as they are.** Costs nothing. *Recommended*: nothing measured needs it.
   - b. **Minify the injected stylesheet in figma's build with esbuild**, which is already a dependency: a few lines, about 15 KB saved, ui's export unchanged.
   - c. **Lightning CSS after PostCSS in figma's build**, minifying and lowering: a new native dependency for 2 KB more than b, and a parity measurement on ui's sheet first.

3. **How should Toghen answer the `light-dark()` lowering that consumers' Lightning CSS steps do today?**
   - *Context*: With Vite 8's defaults, Lightning CSS rewrites `light-dark(a, b)` into flags that exist only where `color-scheme` is declared in the same sheet. Even then, they are decided where the token is declared, not on the subtree that switches. Tailwind turns this lowering off for the same reason.
   - a. **Measure it in a browser, then document it in the guide (exclude `Features.LightDark`), and pin it in the parity test.** Costs units B and C. *Recommended*: the guide's claim then rests on a measurement, and a Lightning CSS release that changes the behaviour fails a test.
   - b. **Document it without measuring.** Faster, and rests on the CSS rule alone.
   - c. **No change.** A consumer finds out from a page with no colour.

4. **If a second adapter is built: how is a path with a number after a dot written?**
   - *Context*: Asked only if the first question is answered b or c. Lightning CSS's tokenizer turns `.500` into the number 0.5 before a visitor sees it, so `t(color.gray.500)`, `t(space.100)` and the documented member path `t(elevation.raised.0.color)` cannot be read. PostCSS keeps the text.
   - a. **Both adapters accept a quoted path,** `t("color.gray.500")`, which a CSS string carries through every tool unchanged. The unquoted form stays for paths without numeric segments, and the Lightning CSS adapter refuses an unquoted path whose numbers it cannot trust, with a sentence. Costs one form in the core and the docs. *Recommended*: standard CSS, the same in both adapters, and no pre-processing.
   - b. **The Lightning CSS adapter reads the source text itself before Lightning CSS parses it.** It works only as a wrapper around `transform`, not as a visitor that Vite or Parcel can compose, and it re-tokenises CSS in JS.
   - c. **Refuse numeric segments in the Lightning CSS adapter.** No new form, but common palettes (`gray.500`) and the documented member paths become unreachable there.

5. **If a second adapter is built: what does it do with mixins?**
   - *Context*: Asked only if the first question is answered b or c. Lightning CSS keeps `@mixin` and `@apply` in place (1.33.0) and expands neither. The CSS mixins polyfill is PostCSS-only. Only Chromium parses mixins, behind a flag. A visitor can expand them (the docs show one), but it meets the open regression and the missing-semicolon bug.
   - a. **Write `@mixin` as the PostCSS adapter does, and leave expansion to the consumer's tools or the browser.** It keeps "Toghen writes the standard form and depends on no polyfill". Costs nothing extra, but components need a PostCSS pass until browsers ship. *Recommended*: the same contract in both adapters, and nothing that chases an unstable draft.
   - b. **Expand `@apply` of Toghen's own mixins inside the adapter, behind an option.** Components work in a Lightning CSS-only pipeline, at the price of Toghen maintaining a partial polyfill of a draft.
   - c. **Leave `@toghen mixins` out of the Lightning CSS adapter,** with a sentence that points to the PostCSS adapter. It says plainly what is missing, and serves a consumer who mixes tools least.

6. **If a second adapter is built: where does the shared spelling live?**
   - *Context*: Asked only if the first question is answered b or c. Lib says two adapters cannot import from each other, and that it "writes nothing out". The value spelling (`valueToCss`, colours, `font`, family quoting) and the directive logic are in postcss today.
   - a. **A core and both shells in one repository and package,** renamed `@toghen/css` with two entry points (`@toghen/css/postcss`, `@toghen/css/lightningcss`). `@toghen/postcss` is kept as a re-export of the first. One test suite runs both shells. Costs a rename and a deprecated alias. *Recommended*: the shells cannot drift apart, with one version and one test run, and no constitution change for lib.
   - b. **Three packages in three repositories**: `@toghen/css` for the core, `@toghen/postcss` and `@toghen/lightningcss`. It follows "one repository per package", at the price of three release trains and cross-repository test runs.
   - c. **Move the value spelling into lib.** Lib's rule "writes nothing out" is amended and the *css* question widens. It serves any future CSS adapter, and puts output spelling in the core.

7. **If a second adapter is built: when may it start, given the open regression?**
   - *Context*: Asked only if the first question is answered b or c. Since 1.30.2, a visitor that returns a rule holding a `var()` fails with "failed to deserialize", and the fix is unmerged. Deleting the `null` members first works on 1.33.0, but re-spells a few values (`1%` as `1.0%`).
   - a. **Wait for a release with the fix,** and build the core extraction meanwhile. No workaround to maintain, but no date. *Recommended*: the core extraction is useful on its own, and a workaround in a published package is debt.
   - b. **Build now with the workaround,** and remove it when the fix ships. Ships sooner, with a known re-spelling and a test that fails when the workaround becomes unnecessary.
   - c. **Pin 1.30.1,** the last release before the regression. No workaround, but it forgoes 1.33.0's fix that keeps `@apply` inside its rule, which Toghen's output needs.

## Answers (23 November 2026)

Egildo, 23 November 2026: "1a 2b 3a".

1. **a.** Lightning CSS is an environment Toghen's output must survive. The adapter stays on PostCSS. A second adapter waits for the four conditions under *Recommendation*. Questions 4 to 7 are not asked.
2. **b.** figma's build minifies the stylesheet it injects into `ui.html` with esbuild, which it already has. ui's exported stylesheet is unchanged.
3. **a.** The `light-dark()` lowering is measured in a browser first. Then the guide tells consumers to exclude it, and the parity test pins it.

### The units, as ruled

Units A to D stand as written above, with two changes.

- **Postcss's base.** Units A and C cut their branch from postcss's `review-fixes`, not `main`, so the review round's fixes are in the goldens. B runs now, and C follows B.
- **A new unit E, for the second answer.**

| Unit | Repository | Worktree folder | Branch | Starts |
|---|---|---|---|---|
| E | figma: minify the injected stylesheet | `figma-minify` | `lightningcss-minify`, from figma's `review-fixes` | Now |

**E. Minify the injected stylesheet.**

- In `figma/build.mjs`, the design system's stylesheet and the plugin's own `ui/styles.css` go through esbuild's CSS minifier (`transform`, `loader: 'css'`, `minify: true`) before they are injected into `ui.html`. Do it in production builds, or always if the tests allow it. Choose one and say why.
- Esbuild's target for the CSS is Figma's Chromium. Make sure it lowers nothing: nesting, `light-dark()`, `oklch()` and `@layer` must stay as written. Check the output for each.
- `scripts/check-leaks.mjs` and every test that reads `dist/` or the stylesheet's text must still pass. Change a test only where it read whitespace and nothing that matters, and say which.
- Measure `ui.html` before and after, and report the size.
- Open the built plugin's screen in the dev preview (`pnpm preview`) and check that it renders, in both themes, as before.

