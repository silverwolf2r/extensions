# Fic Reader — source catalog

Installable **source configs** for the Fic Reader app. Everything here is pure
**data** (URLs + declarative parse rules); the app ships the engine, these files
point it at sites. Add one in the app via **Browse → (Books / Sources) → add
from URL**, pasting the file's **raw** URL.

## Catalogs

### Fiction sources — `fiction-sources.json`
Archive of Our Own, Royal Road, FanFiction.Net, Wattpad. Rung-3 DSL format
(`schema: 3`, capability pipelines) used by the **aggregator-shell** app. Search
+ reading (details / chapters / content) run entirely on-device.

```
https://raw.githubusercontent.com/silverwolf2r/extensions/main/fiction-sources.json
```

> The aggregator-shell app already **bundles** these four by default; this file
> is the canonical published copy (for updates, review, and forks).

### Book sources — `booksources.json`
Library Genesis, Anna's Archive, Z-Library, VK, NovelFlow. Provider-strategy
format (`parser: html-table | flex-rows | web-cards | web-search | chapter-walk`).
Never bundled — **user-added only.**

```
https://raw.githubusercontent.com/silverwolf2r/extensions/main/booksources.json
```

## Authoring

See [`authoring-a-source.md`](./authoring-a-source.md) for how to write and test a
source, the two hard requirements a site must meet, and worked examples
(`examples/`).

## Formats note

Two schemas exist for historical reasons: the newer rung-3 DSL
(`fiction-sources.json`) and the older provider strategies (`booksources.json`).
They're gradually converging on the rung-3 DSL — see the app's
`docs/aggregator-shell-overview.md`.
