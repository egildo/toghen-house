# Toghen

Toghen is a design-tokens toolkit. It reads one portable token source, written in the [W3C Design Tokens format](https://www.w3.org/community/reports/design-tokens/CG-FINAL-format-20251028/) and [Resolver](https://www.w3.org/community/reports/design-tokens/CG-FINAL-resolver-20251028/), and turns it into what each target needs: CSS at build time, and Figma variables, styles and component sets. Toghen is general-purpose. It holds no knowledge of any one design system.

## The model

> A component is a group, it supports its subgroups, a part re-points the group; worlds are everything above it.

[The component model](https://codeberg.org/Toghen/lib/src/branch/main/model.md) says what each word means, who sets each choice, and how a choice reaches CSS and Figma.

## Packages

| Package | What it does |
|---|---|
| [`@toghen/lib`](https://codeberg.org/Toghen/lib) | The core engine: tokens, groups, resolution, the Resolver, and queries across modifiers. Adapters translate its output and never redefine it. |
| [`@toghen/postcss`](https://codeberg.org/Toghen/postcss) | Resolves tokens in stylesheets at build time, writes contexts as cascade layers, and writes each component as mixins. |
| [`@toghen/figma`](https://codeberg.org/Toghen/figma) | Syncs tokens to Figma variables and styles, builds component sets from component files, and audits and exports them. The `toghen-figma` command runs a whole sync from a repository. |
| [`@toghen/query`](https://codeberg.org/Toghen/query) | A command line that asks the core a question and prints the answer, one command for each of the core's nine questions, each answer one call to one export. |
| [`toghen-ui`](https://codeberg.org/Toghen/ui) | The design system of the Figma plugin's screen, and Toghen's playground. |
| [`toghen-skills`](https://codeberg.org/Toghen/skills) | A Claude Code plugin marketplace: skills for the pipeline from a repository to Figma and FigCraft, and for each app at a pinned version. No code. |
| [`gosfuig`](https://codeberg.org/Toghen/gosfuig) | A perceptual (OKLCH) palette generator, with contrast checks and W3C design-token export. |

## A small example

Here is a stylesheet, with the running example's document `ds` as its tokens:

```css
@layer tokens {
    @toghen contexts;
}

@toghen mixins;

.button {
    display: inline-flex;
    @apply --button;
}

.button-icon {
    @apply --button-icon;
}
```

The PostCSS plugin writes the `.button` rule like this:

```css
.button {
    display: inline-flex;
    --_button-background-color: initial;
    --_button-color: initial;
    background-color: var(--_button-background-color, var(--button-background-color, #ffffff));
    color: var(--_button-color, var(--button-color, #1a1a1a));
    padding-block: var(--button-padding-block, 8px);
```

[The tutorial](https://codeberg.org/Toghen/postcss/src/branch/main/tutorial.md) builds this from the tokens, one step at a time.

## Where the docs are

Each package has a guide for tasks, a reference for the API, specs for how it works and why, and decisions for dated rulings and open questions. Read the component model first, then the guide of the package you use.

| Package | Guide | Reference | Specs | Decisions | Also |
|---|---|---|---|---|---|
| `@toghen/lib` | [guide](https://codeberg.org/Toghen/lib/src/branch/main/guide.md) | [reference](https://codeberg.org/Toghen/lib/src/branch/main/reference.md) | [specs](https://codeberg.org/Toghen/lib/src/branch/main/specs.md) | [decisions](https://codeberg.org/Toghen/lib/src/branch/main/decisions.md) | [component model](https://codeberg.org/Toghen/lib/src/branch/main/model.md) |
| `@toghen/postcss` | [guide](https://codeberg.org/Toghen/postcss/src/branch/main/guide.md) | [reference](https://codeberg.org/Toghen/postcss/src/branch/main/reference.md) | [specs](https://codeberg.org/Toghen/postcss/src/branch/main/specs.md) | [decisions](https://codeberg.org/Toghen/postcss/src/branch/main/decisions.md) | [tutorial](https://codeberg.org/Toghen/postcss/src/branch/main/tutorial.md), [measurements](https://codeberg.org/Toghen/postcss/src/branch/main/measurements/README.md) and [results](https://codeberg.org/Toghen/postcss/src/branch/main/measurements/RESULTS.md) |
| `@toghen/figma` | [guide](https://codeberg.org/Toghen/figma/src/branch/main/guide.md) | [reference](https://codeberg.org/Toghen/figma/src/branch/main/reference.md) | [specs](https://codeberg.org/Toghen/figma/src/branch/main/specs.md) | [decisions](https://codeberg.org/Toghen/figma/src/branch/main/decisions.md) | [component files](https://codeberg.org/Toghen/figma/src/branch/main/component-files.md), [screen](https://codeberg.org/Toghen/figma/src/branch/main/ui.md), [UX map](https://codeberg.org/Toghen/figma/src/branch/main/ux-map.md), [UX rulebook](https://codeberg.org/Toghen/figma/src/branch/main/ux-rulebook.md), [UX smell log](https://codeberg.org/Toghen/figma/src/branch/main/ux-smell-log.md) |
| `@toghen/query` | [guide](https://codeberg.org/Toghen/query/src/branch/main/guide.md) | None: it exposes transcripts, not an API | [specs](https://codeberg.org/Toghen/query/src/branch/main/specs.md) | [decisions](https://codeberg.org/Toghen/query/src/branch/main/decisions.md) | [sessions](https://codeberg.org/Toghen/query/src/branch/main/sessions/) |
| `toghen-ui` | [guide](https://codeberg.org/Toghen/ui/src/branch/main/guide.md) | None: it exposes documents, not an API | [specs](https://codeberg.org/Toghen/ui/src/branch/main/specs.md) | [decisions](https://codeberg.org/Toghen/ui/src/branch/main/decisions.md) | |
| `toghen-skills` | [guide](https://codeberg.org/Toghen/skills/src/branch/main/guide.md) | None: it holds skills, not code | [specs](https://codeberg.org/Toghen/skills/src/branch/main/specs.md) | [decisions](https://codeberg.org/Toghen/skills/src/branch/main/decisions.md) | |

Contributors and AI agents: [CLAUDE.md](./CLAUDE.md) says which of these bind you, and the invariants that keep Toghen general-purpose.

## Build and test

Each repository builds and tests on its own. In a clone of one of the five code packages:

```sh
pnpm install
pnpm check
```

`pnpm check` is the gate CI runs: Biome, the type check, the tests, the build and the purity check. house and skills hold no package; their gate is the purity check alone (`node shared/check-purity.mjs` in house, `node scripts/check-purity.mjs` in skills).

To work across packages on one machine, put the checkouts in one folder and copy house's `workspace/` files into it. That folder is not a repository: `pnpm install` there links lib, postcss, query, ui and figma to each other, and `pnpm build`, `pnpm typecheck` and `pnpm test` there run across all five.

In a clone of figma, `pnpm build` writes the plugin into `dist/`. [Sync a repository into a Figma file](https://codeberg.org/Toghen/figma/src/branch/main/guide.md#from-the-repository-to-figma) walks through importing the plugin and the first sync. To see the plugin's screen in a browser, run `pnpm preview` there after the build. See [the preview page](https://codeberg.org/Toghen/figma/src/branch/main/ui.md#the-preview-page).

A shell may resolve `node` to an old version and fail without a clear message. Put Node 24 first on your `PATH`, the version `.node-version` names.

## License

MIT. See [LICENSE](./LICENSE).
