---
name: obsidian-graph
description: Configure Obsidian Graph view — color groups, filters, forces, display settings, palette choice. Use whenever the user wants to change how the graph looks or what it shows, including coloring by folder/category/topic/tag, hiding noise via filter, adjusting forces (repel/link distance), or picking a palette. Triggers include "configure the graph", "color the graph", "fix the graph", "show/hide tags on the graph", "exclude folder from graph", "graph.json", "color groups", "graph filter", "graph forces", "graph palette", "make graph readable", "spread the graph out", "highlight hubs on the graph", "why is the graph one color".
---

# Obsidian Graph view

Global graph configuration lives in `.obsidian/graph.json`. Local graph (Graph view opened from a single note) lives in `.obsidian/workspace.json` and **does not inherit** from global. This skill covers global graph unless explicitly noted.

For diagnosing *what's wrong* with a graph (over-connection, render-only edges, false categories), use the paired `obsidian-graph-audit` skill first.

## Workflow when the user asks "configure my graph"

1. **Ask or read intent first.** What's bothering them, or what do they want to see? Color groups without a goal are pure aesthetics — skip them.
2. **⚠️ Have the user close Graph view (or quit Obsidian).** This is mandatory — Obsidian rewrites `graph.json` when Graph view closes, wiping external edits. See Caveats.
3. **Read the current `.obsidian/graph.json`.** Don't reset settings the user has tuned (especially `scale`).
4. **Measure vault size:** `find . -name '*.md' -not -path './.obsidian/*' -not -path './.trash/*' -not -path './.claude/*' | wc -l`. Forces depend on this.
5. **Apply minimum changes**, explain each. 14 color groups on 600 notes is too many — 3-8 is the proven range.
6. **Keep only what adds signal.** Filter > group when you don't want to see something; group > filter when you want to differentiate.
7. **Verify decimal RGB math.** A wrong byte in a color value silently renders as a different color. Always round-trip HEX↔decimal (see below).
8. **Have the user open Graph view after edits.** Changes take effect on next open.

## Full schema of `graph.json`

20 fields. Source: [community gist on syncing graph settings](https://gist.github.com/bobheadxi/ad4bc77a7b8c80d26f7668dac8a47576), [example repo](https://github.com/GrangbelrLurain/.obsidian/blob/master/graph.json). Obsidian has no official schema document.

### Filters block

| Field | Type | Purpose |
|---|---|---|
| `collapse-filter` | bool | Whether the Filters block is collapsed in the UI |
| `search` | string | Filter query (same syntax as core Search) |
| `showTags` | bool | Tag nodes shown as separate nodes |
| `showAttachments` | bool | Files (PNG, PDF) shown as nodes |
| `hideUnresolved` | bool | Hide unresolved wikilinks |
| `showOrphans` | bool | Show nodes with no connections |

### Color groups block

| Field | Type | Purpose |
|---|---|---|
| `collapse-color-groups` | bool | Whether the Groups block is collapsed |
| `colorGroups` | array | Array of `{query, color}` objects |

### Display block

| Field | Type | Purpose |
|---|---|---|
| `collapse-display` | bool | Whether the Display block is collapsed |
| `showArrow` | bool | Show direction arrows on edges |
| `textFadeMultiplier` | number | 0 = labels never visible; 1 = always; ~0.5 = visible at moderate zoom |
| `nodeSizeMultiplier` | number | 1 = default; >1 makes hubs bigger |
| `lineSizeMultiplier` | number | 1 = default; <1 makes edges thinner |

### Forces block

| Field | Type | Purpose |
|---|---|---|
| `collapse-forces` | bool | Whether the Forces block is collapsed |
| `centerStrength` | number | Pull toward center; 0.5 = sane default |
| `repelStrength` | number | Node repulsion; 10 default, 15-25 for larger vaults |
| `linkStrength` | number | Edge spring stiffness; 0.7-1 |
| `linkDistance` | number | Target edge length; 150-250 |

### Other

| Field | Type | Purpose |
|---|---|---|
| `scale` | number | Current zoom — DO NOT edit by hand; Obsidian overwrites |
| `close` | bool | Was the view closed; don't touch |

## What creates a graph edge

Before configuring color groups or filters, understand which references in a note become edges in the graph. Obsidian renders an edge from any of the following:

- `[[Wikilinks]]` in the body of a note
- Wikilinks inside frontmatter list fields, including `topics:` and `categories:` (any list-of-wikilinks pattern)
- **Wikilinks inside single-value frontmatter properties** — e.g. `author: "[[J. R. R. Tolkien]]"`, `director: "[[Christopher Nolan]]"`, `host: "[[Tim Ferriss]]"`. These generate graph edges identical to wikilinks in the body. This is often missed.
- Tag nodes when `showTags: true` (otherwise tags don't appear at all)

What does NOT create an edge:
- Plain string properties (`role: "Senior Engineer"`)
- Tags (when `showTags: false`)
- Wikilinks inside fenced code blocks (` ```...``` `) or inline code (`` ` ``)

This matters when you design color groups: a note with `author: "[[J. R. R. Tolkien]]"` is already linked to the `J. R. R. Tolkien` node, so an additional `tag: tolkien` adds nothing graph-wise. Similarly, a `topics:` entry is redundant if a property field already references the same target. See `obsidian-graph-audit` Pattern H.

## Search syntax (for `search` filter and `query` in color groups)

Same syntax as [core Search](https://obsidian.md/help/Plugins/Search).

### Operators

| Operator | Matches | Example |
|---|---|---|
| `path:"folder"` | Path contains string | `path:"Projects/Active"` |
| `file:"name"` | File name | `file:"Home"` |
| `tag:#name` | Note has tag | `tag:#hub` |
| `line:(query)` | Same line | `line:(foo bar)` |
| `block:(query)` | Same block | `block:(query)` |
| `section:(query)` | Same heading section | `section:(Notes)` |
| `task:(query)` / `task-todo:` / `task-done:` | Tasks | `task-todo:(review)` |
| `content:(query)` | Body only, not frontmatter | `content:(meeting)` |
| `["property"]` | Note has property (substring, not strict — see caveat) | `["status"]` |
| `["property":"value"]` | Property contains value (substring) | `["categories":"Companies"]` |
| `/regex/` | Regex pattern | `/^Daily/` |
| `match-case:` / `ignore-case:` | Toggle case sensitivity | |

### Logic

| | |
|---|---|
| `-foo` | NOT |
| `foo bar` (space) | AND |
| `foo OR bar` | OR |
| `"phrase"` | exact phrase |
| `(foo OR bar)` | grouping |

### Caveats for color groups

- **Order matters.** First matching group wins. Tags first (semantic role), categories next (entity type), topics third (epic clusters), paths last (catch-all).
- **`["property":"value"]` is substring match, not strict.** `["categories":"Recipe"]` matches both `Recipes` and `Recipe Drafts`. Mitigation: use **distinct strings** as category/topic names. If strict match is needed, convert to a tag.
- **Regex is supported** — `/^Daily/`, `/202\d-/`, etc.

## Color group format

```json
{
  "query": "<search query>",
  "color": { "a": 1, "rgb": <decimal RGB> }
}
```

- `a` — alpha (transparency), 0 = invisible, 1 = solid. Almost always 1.
- `rgb` — 24-bit decimal: `R*65536 + G*256 + B`.

### HEX → decimal conversion

```bash
python3 -c "h='E07B5A'; print(int(h, 16))"  # → 14711642
```

Or manually: `0xE07B5A` = `224*65536 + 123*256 + 90` = `14,711,642`.

**One wrong byte renders as a different color.** Always round-trip:

```bash
python3 -c "n=14711642; print(f'#{n:06X}')"  # → #E07B5A
```

### Palette — pastel set

Soft, calm overall. Luminosity values are close, so 5+ groups in this set may visually merge.

| Color | HEX | Decimal | Use |
|---|---|---|---|
| Coral | `#E07B5A` | 14711642 | Active priority project |
| Pink | `#E36B9B` | 14904219 | Personal / relationships |
| Purple | `#9B6BE3` | 10185699 | Habit / personal logs |
| Sky blue | `#5AAFE0` | 5943264 | Learning |
| Green | `#7BC15A` | 8110426 | Lifestyle / household |
| Gold | `#E0C95A` | 14731610 | Tech / sandbox |
| Teal | `#5EC1B5` | 6209973 | Planning |
| Yellow | `#F0D060` | 15781984 | Hubs / MOCs |
| Cream | `#D9C89E` | 14272670 | Zettelkasten / passive |
| Mustard | `#C9A03B` | 13213755 | External entities |
| Soft gray | `#8B9DA9` | 9149865 | Clippings / refs |
| Muted blue-gray | `#708FA8` | 7376808 | Daily |
| Neutral gray | `#A0A0A0` | 10526880 | Inputs / rough |
| Dim gray | `#666666` | 6710886 | Tag underlay |

### Palette — high-contrast set

Use when 5+ pastel groups blur together. Luminosity ranges from bright (hubs) to dark (Daily) — the eye separates them.

| Color | HEX | Decimal | Use |
|---|---|---|---|
| Bright Gold | `#E8B931` | 15252785 | Hubs (primary accent) |
| Coral red | `#D9583E` | 14244414 | Active project (catch-all) |
| Mustard | `#B8861F` | 12092447 | Entity type (Companies) |
| Light Purple | `#B47AC9` | 11828425 | Entity type (Contacts) |
| Deep Purple | `#7E4FA8` | 8278440 | Entity type (Concepts, MOCs) |
| Saturated Green | `#6FA855` | 7317077 | Lifestyle category |
| Bright Cyan | `#2EB5C9` | 3061193 | Other projects (catch-all) |
| Cream | `#D9C89E` | 14272670 | Passively saved (Clippings, Meetings) |
| Soft gray | `#8B9DA9` | 9149865 | Reports / refs |
| Dark Slate | `#4A5C6B` | 4873323 | Daily / archive |

## Four coloring strategies

Color groups can target four different dimensions of a note. Different vaults call for different strategies; mature vaults usually mix them.

### A. Folder-based (path:)

```json
{"query": "path:\"Projects/Active\"", ...}
```

**When to use:** Project-organized vault where the folder *is* a meaningful unit. Each project lives in its own folder, the layout is stable.

Pros:
- Simple mental model — "everything in this folder"
- No frontmatter required
- New files inherit the color automatically

Cons:
- Doesn't capture **cross-cutting concerns** (hub-pages, status workflows) that span folders
- Reorganizing folders breaks the coloring
- Files in one folder may be of different types (an epic and a routine note get the same color)

### B. Category-based (`["categories":"X"]`)

```json
{"query": "[\"categories\":\"Companies\"]", ...}
```

**When to use:** Kepano-style vault with frontmatter `categories:`. Category = entity type (company, contact, recipe, film). Files in one category are semantically homogeneous.

Pros:
- Color follows **entity meaning**, not accidental location
- A file can belong to multiple categories (multi-color via group order)
- Folder reorganization doesn't break it

Cons:
- **Property-search is substring match.** `["categories":"Recipe"]` matches both `Recipes` and `Recipe Drafts`. Categories must be **distinct strings** (see Caveats)
- Requires disciplined frontmatter
- Files without categories (rough notes, drafts) are uncolored

### C. Topic-based (`["topics":"X"]`)

```json
{"query": "[\"topics\":\"Active Initiative\"] OR file:\"Active Initiative\"", ...}
```

**When to use:** Kepano-style with `topics:` fields (wikilinks to epics or coordinating concepts). Useful for an active project with epic-coordinators and many files attached to each.

Pros:
- Color = semantic cluster (the epic and everything tied to it)
- Cross-folder: files from `Reports/`, `Interview Prep/`, `BootCamp/` can land in one color if they share a topic
- **Ghost nodes work** — the query reads the topic from frontmatter; it doesn't need the epic file to exist
- Sharper than path catch-all for a large active project (splits work streams visually)

Cons:
- Same substring-match caveat as `["categories":...]`
- Files without topics aren't colored
- `OR file:"X"` is needed to color the epic node itself (the epic doesn't list itself as its own topic)
- More groups means closer to the 14-group practical ceiling

### D. Tag-based (`tag:#name`)

```json
{"query": "tag:#hub", ...}
```

**When to use:** Tag taxonomy for **cross-cutting concerns** — hubs (`#hub`), statuses (`#status/done`), types (`#moc`, `#evergreen`). A tag describes "what role does this file play in the graph", not "what is this file".

Pros:
- Strict match (not substring, unlike property)
- Multi-dimensional — tags cross folders and categories
- Easy to add or remove ad hoc

Cons:
- Tag taxonomy can sprawl without discipline
- A bare tag with no file is an orphan node when `showTags: true`
- Bureaucratic for very large vaults

### Mixed approach — typical for mature vaults

```json
"colorGroups": [
  {"query": "tag:#hub", ...},                                                      // cross-cutting accent
  {"query": "[\"categories\":\"Companies\"]", ...},                                 // entity types
  {"query": "[\"categories\":\"Contacts\"]", ...},
  {"query": "[\"topics\":\"Active Initiative\"] OR file:\"Active Initiative\"", ...}, // epic clusters
  {"query": "[\"topics\":\"Other Initiative\"] OR file:\"Other Initiative\"", ...},
  {"query": "path:\"Daily\"", ...},                                                 // structural folder
  {"query": "path:\"Projects\"", ...}                                               // catch-all
]
```

Order matters — tags first (semantic role), categories second (entity type), topics third (epic clusters of the active project), paths last (catch-all). First match wins.

### Choosing a strategy

| Signal | Strategy |
|---|---|
| Projects in `Projects/X/`, little frontmatter | Folder-based |
| Kepano-style, `categories: [[X]]` everywhere | Category-based |
| Active project with epics, `topics:` on coordinators | Topic-based |
| Goal: highlight hubs / statuses / workflow stages | Tag-based |
| 600+ notes, active project + entity types + epics + hubs | Mixed (all four together) |

## Best practices

1. **Color by meaning, not by folder.** Folder is one dimension; see the four-strategy section above. Pick by structure, often combine.

2. **3-8 color groups, not 14.** Each group is render overhead and visual noise. If 14 feels needed, the user probably wants 5 conceptual buckets, not 14 individual ones.

3. **Order matters.** Specific queries first, broad last. First match wins.

4. **Filter only for active noise, not for periphery.** Folders that are weakly connected (rough notes, drafts, scratch) form their own peripheral cluster and don't crowd the center. Filter is justified when nodes from a folder *embed* into the center and pollute important clusters — typical candidates: `Templates` (if not in `userIgnoreFilters`), archive tagged `#archive`. Before adding `-path:` or `-tag:`, ask: "are these nodes currently in the center, or on the edge?". On the edge — leave them, they give visual segregation.

5. **Don't show tags on the graph unless they're cross-cutting.** Tags isolated to a single note become satellite-nodes with no signal. Tags that duplicate folder names add tangle without meaning. Enable `showTags: true` only when tags form a real semantic network (`#moc`, `#status/*`, thematic concepts).

6. **`textFadeMultiplier 0` means labels never visible.** If you want to read names at zoom-in, set ≥ 0.3 (0.5 is a good default).

7. **Forces are vault-size dependent.** Sane defaults for ~600 notes: `centerStrength: 0.5`, `repelStrength: 15-25`, `linkStrength: 0.7-1`, `linkDistance: 150-250`. Larger vault → more repel + shorter links.

8. **Don't edit `scale` by hand.** Obsidian overwrites it on every zoom. Manual edits are wiped.

9. **Local graph is configured separately.** Local graph config lives in `.obsidian/workspace.json` under workspace state; it doesn't inherit from global. For shared color groups locally, install the community plugin **Sync Graph Settings**.

10. **CSS snippet for defaults, color groups for exceptions** (community pattern). Set base node/tag/attachment/unresolved colors via a CSS snippet once, use color groups only for specific subsets. Less retina-burning than 8 path-based groups doing everything.

## CSS classes (for snippets)

The graph renders to canvas/WebGL, but Obsidian reads CSS color rules and converts them. Use `!important` to override. Path: Settings → Appearance → CSS snippets.

| Class | What it colors |
|---|---|
| `.graph-view.color-fill` | Default node fill |
| `.graph-view.color-fill-focused` | Focused node |
| `.graph-view.color-fill-tag` | Tag node (theme-dependent: `.theme-dark` / `.theme-light`) |
| `.graph-view.color-fill-attachment` | Attachment |
| `.graph-view.color-fill-unresolved` | Broken link |
| `.graph-view.color-fill-highlight` | Hover |
| `.graph-view.color-arrow` | Direction arrow |
| `.graph-view.color-circle` | Node ring |
| `.graph-view.color-line` | Edge |
| `.graph-view.color-line-highlight` | Hover edge |
| `.graph-view.color-text` | Node label |

`#HEX`, `rgb()`, and `rgba()` are all supported. For transparent edges in graph view, use rgba.

## Recipes

### Hide noise

```json
"search": "-path:\"Templates\" -path:\"Rough Notes\" -tag:#archive"
```

### Focus on one project

```json
"search": "path:\"Projects/Active\""
```

### Color by status (tag-based)

```json
"colorGroups": [
  {"query": "tag:#status/done",  "color": {"a": 1, "rgb": 8110426}},
  {"query": "tag:#status/draft", "color": {"a": 1, "rgb": 14731610}},
  {"query": "tag:#status/seed",  "color": {"a": 1, "rgb": 9149865}}
]
```

### Color by project (folder-based)

```json
"colorGroups": [
  {"query": "path:\"Projects/A\"", "color": {"a": 1, "rgb": 14711642}},
  {"query": "path:\"Projects/B\"", "color": {"a": 1, "rgb": 10185699}},
  {"query": "path:\"Notes\"",       "color": {"a": 1, "rgb": 14272670}},
  {"query": "path:\"Daily\"",       "color": {"a": 1, "rgb": 7376808}}
]
```

### Color by category (Kepano-style)

```json
"colorGroups": [
  {"query": "[\"categories\":\"Companies\"]", "color": {"a": 1, "rgb": 12092447}},
  {"query": "[\"categories\":\"Contacts\"]",  "color": {"a": 1, "rgb": 11828425}},
  {"query": "[\"categories\":\"Recipes\"]",   "color": {"a": 1, "rgb": 7317077}},
  {"query": "[\"categories\":\"Reports\"]",   "color": {"a": 1, "rgb": 9149865}}
]
```

### Color by topic (epic clusters of an active project)

```json
"colorGroups": [
  {"query": "[\"topics\":\"Initiative A\"] OR file:\"Initiative A\"",
   "color": {"a": 1, "rgb": 14904219}},
  {"query": "[\"topics\":\"Initiative B\"] OR file:\"Initiative B\"",
   "color": {"a": 1, "rgb": 5943264}}
]
```

`OR file:"X"` is needed because the epic node's own `topics:` references upstream parents, not itself.

### Highlight hubs / MOCs

Tag-based — strict, recommended:

```json
{"query": "tag:#hub", "color": {"a": 1, "rgb": 15252785}}
```

Folder/file-based fallback (when tags aren't in use):

```json
{"query": "file:\"Home\" OR file:\"About\" OR path:\"Categories\"",
 "color": {"a": 1, "rgb": 15781984}}
```

### Hide infrastructure and archive via tag taxonomy

Establish: `#dashboard` for view pages (kanban, metrics, charts), `#no-graph` for logs / setup docs / configs (pure control tag), `#archive` for archived content.

```json
"search": "-tag:#dashboard -tag:#no-graph -tag:#archive"
```

Combine with category-exclude to hide entity types not wanted on the conceptual graph:

```json
"search": "-tag:#dashboard -tag:#no-graph -[\"categories\":\"Vacancies\"]"
```

### Forces for a 600+ note vault

```json
"centerStrength": 0.5,
"repelStrength": 22,
"linkStrength": 0.7,
"linkDistance": 220,
"textFadeMultiplier": 0.5,
"nodeSizeMultiplier": 1.4,
"lineSizeMultiplier": 0.8
```

## Caveats and gotchas

- **⚠️ Obsidian rewrites `graph.json` when Graph view closes.** When the graph tab closes (or the app quits), Obsidian flushes its in-memory state to the file, overwriting external edits. Symptom: just-made edits are gone, `close: true`, and `colorGroups: []`.

   **Procedure:** either (a) Cmd+Q quit Obsidian → edit `graph.json` → relaunch → open Graph view; or (b) close the Graph view tab → edit → `Cmd+P → Reload app without saving` → open Graph view. **Never edit while Graph view is open.**
- **Property-search is substring, not strict.** `["topic"]` matches `topic`, `parent_topic`, and `topic_alternative`. Don't use property search for color groups when precision matters; use tags instead.
- **Many color groups slow rendering.** Each group is `O(notes × groups)` per repaint. Above 15 groups it's noticeable on large vaults.
- **`showOrphans: false` hides early sessions of a new vault.** In a fresh vault most notes are orphans; without `showOrphans: true` you see an empty screen.
- **Filter cascading.** Removing a note via filter also removes its edges, and its neighbors may become orphans on the graph.
- **Extended Graph plugin** (ElsaTam/obsidian-extended-graph, locally-installed) gives deeper config (per-tag colors, edge weights). Recommend it when core hits a ceiling — don't try to squeeze more from the core graph.

## Multiple views / saved presets

Core Obsidian doesn't support saved graph presets — one global config, one local. To switch between multiple views, install a plugin:

| Plugin | What it does | Install | When to pick |
|---|---|---|---|
| **Extended Graph** (ElsaTam) | Multiple views + switching, per-tag colors, image nodes, advanced filters, SVG export | Community store | When you want both presets and a stronger graph overall |
| **Graph Presets** (ycnmhd) | Markdown-based: preset = `.md` with YAML, apply / update from active graph | BRAT (not in community store) | When you want presets as notes in the vault, versioned in git |
| **Persistent Graph** | Saves node positions + filters + workspaces | Community store | When fixing the layout matters, not just settings |
| **Sync Graph Settings** | Sync global → local color groups | Community store | When you use local graph often; **not for presets** |

**Don't recommend a plugin preemptively.** Have the user describe the views they want to switch between (e.g. "focused project view" vs "full vault"). If there's no concrete use case, one global config is enough.

**Manual approach without a plugin**: store variants of `graph.json` in `.obsidian/graph-presets/` and swap with a script. Not recommended — no UI for switching, easy to forget to restore, and races with Obsidian's overwrite (see Caveats).

## When NOT to touch the graph

- The graph is used for 5-10 minutes a day for serendipity, not as primary navigation. Don't optimize it as a dashboard.
- The user hasn't said what's bothering them. Don't propose edits unprompted.
- Aesthetic edits without a clear task = bikeshedding. Get the goal first ("I want to see X / I don't want to see Y"), then config.
