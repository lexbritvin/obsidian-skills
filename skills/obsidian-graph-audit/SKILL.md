---
name: obsidian-graph-audit
description: Diagnose Obsidian Graph view health and clean up over-connection — identify render-only edges (dashboards as topics, log files), mega-categories, broken refs, and false categories that are inflating the graph. Use whenever the user reports the graph feels tangled, overconnected, hairball-like, or noisy, even if they don't explicitly say "audit". Triggers include "why is my graph a hairball", "audit my graph", "graph cleanup", "find graph noise", "too many connections", "graph feels tangled", "what shouldn't be on the graph", "my graph is overconnected", "graph is a mess", "what to exclude from the graph", "why are unrelated notes connected".
---

# Obsidian Graph Audit

A diagnostic workflow for an Obsidian graph that feels tangled, overconnected, or visually noisy. Operates well on Kepano-style vaults (with `categories:` and `topics:` in frontmatter) but the patterns generalize.

This skill **diagnoses and recommends**. It does not silently apply fixes. The deliverable is a report — pain points, ranked by impact, with proposed actions the user can approve, modify, or skip. Mitigation scripts exist (later in this file) but only run after the user explicitly confirms a specific action. Color tweaks and filter changes are out of scope — see the `obsidian-graph` skill for styling.

## When to use

User says:
- "Graph is a hairball / spaghetti / mess"
- "Too many connections, can't read it"
- "Why is X linked to Y? They aren't related"
- "Help me clean up the graph"
- "Find what's making the graph noisy"
- "What should I exclude from the graph?"

## Prerequisites

The diagnostic one-liners below use `rg` (ripgrep), `awk`, `find`, and `python3`. All are common on macOS and Linux except `rg`, which usually requires `brew install ripgrep` or equivalent. If `rg` is not available, substitute `grep -c` (for counts) or `grep -E` (for patterns) — slightly slower on large vaults but otherwise equivalent.

## Core insight

A node draws visual attention proportional to its **degree** (number of edges). High-degree nodes are one of:

- **Real semantic hubs** — coordinating concepts (epics, MOCs, topics-of-many-notes). Keep.
- **Render-only hubs** — accumulators of inbound links for view rendering (dashboards, log files with inline-fields). Remove from graph or convert.
- **Mega-categories** — categories used as "tag for files in this folder" rather than entity types. Refactor.
- **Broken refs** — placeholders, malformed wikilinks creating ghost nodes. Fix.

Audit's job is to classify each visible hub into one of those buckets.

## Workflow

The audit is two phases: **diagnose + recommend** (steps 1–5), then **act on confirmation** (step 6). Never skip phase 1. Never run phase 2 without an explicit go-ahead from the user on the specific action.

### 1. Establish baseline

```bash
find . -name "*.md" -not -path "./.obsidian/*" -not -path "./.trash/*" -not -path "./.claude/*" | wc -l
```

Vaults under ~200 notes don't accumulate the patterns this skill targets. Skip the audit; the graph is just sparse.

### 2. List top out-degree files

```bash
find . -name "*.md" -not -path "./.obsidian/*" -not -path "./.trash/*" -not -path "./.claude/*" -print0 | \
  while IFS= read -r -d '' f; do
    count=$(rg -c '\[\[' "$f" 2>/dev/null || echo 0)
    echo "$count	$f"
  done | sort -rn | head -20
```

Approximate — counts wikilinks anywhere in the file, including inside code blocks. Good enough for triage; verify suspect files individually.

### 3. List top in-degree (most-linked-to) targets

Frontmatter-only count of `topics:` references (more accurate than free-text grep):

```bash
find . -name "*.md" -not -path "./.obsidian/*" -not -path "./.trash/*" -not -path "./.claude/*" -print0 | \
  while IFS= read -r -d '' f; do
    awk 'BEGIN{s=0} /^---$/{s++; next} s==1 && /^topics:/{p=1; next} s==1 && p && /^[a-zA-Z_]+:/{p=0} s==1 && p' "$f" | \
      rg -o '\[\[([^\]|#]+)' -r '$1'
  done | sort | uniq -c | sort -rn | head -20
```

The state-counter (`s++`) is critical: parsing only between the first and second `---` avoids walking into body horizontal rules and miscounting.

Same idea for categories — replace `topics:` with `categories:` in the awk pattern.

### 4. Classify each top hub

For each node in the top-20:

| Type | Signature | Recommended action |
|---|---|---|
| Real semantic hub (epic, MOC, topic) | Has its own file, file is short and coordinates other notes via lists/links | Keep |
| Index page for a category (Recipes.md, Companies.md) | Hub-file with `tag: hub` listing members | Keep |
| Dashboard / view page | Renders queries (Dataview, Bases, charts); inbound topics are render-only | Tag `#dashboard`, exclude |
| Log file with inline-fields | Append-only file with `[key:: value]` bullets; consumed by Dataview | Tag `#no-graph`, exclude |
| Mega-category | Category applied to all files in one folder; folder ≈ category | Convert to topic |
| False category | Project/folder name used as category | Remove |
| Broken placeholder | Template-leaked node like `[[<X>]]` or `[[{{Y}}]]` | Edit out (verify first — may be inside a code block, false alarm) |

### 5. Present the report (always)

Deliver to the user before any action:

1. **Pain points** — the top-degree nodes that are not real semantic hubs, classified into the table above. State *what* the problem is in one sentence per finding.
2. **Recommendations** — for each pain point, propose an action with the expected impact ("strip topic `Dashboard - X` from 26 files, removes 26 edges from a 50-edge fake hub").
3. **Counts and scope** — how many files each action touches, which folders are affected.
4. **Open questions** — anything ambiguous the user must decide (e.g. a tag like `john` that maps to multiple entities; a meta-category that could be soft-hidden or hard-removed).
5. **Action plan, ordered by impact** — high-leverage fixes first, low-cost cleanups later.

Format the report so the user can scan it and reply "do A and C, skip B, ask me again about D". Do not lump actions together — each one is a separate go/no-go decision.

### 6. Act on confirmation only

Only after the user explicitly approves a specific action, run the corresponding mitigation pattern (see Mitigation patterns below) with `--dry-run` first, show the dry-run output, then `--apply`. If the user approved A but the dry-run shows it also touches files outside the expected scope, pause and report — do not proceed on the prior approval.

## Diagnostic patterns

### A. Mega-category from folder-as-category

A category is applied to nearly every file in one folder, accumulating 30+ inbound edges to the category page.

**Signature:** `categories: [[X]]` on N files where N is comparable to the folder's file count, and the category name matches the folder name.

**Why noise:** Files already share the folder's structural identity. The category-as-page collects them into a star pattern that says nothing the path didn't already.

**Fix:** Demote the category to a topic. `categories:` is for *entity type* ("this is a Company"); `topics:` is for *thematic association* ("this is about Onboarding Course"). Bulk-rewrite the frontmatter.

**Diagnostic:**
```bash
# Files with category X grouped by parent folder
CAT="<CategoryName>"
find . -name "*.md" -not -path "./.obsidian/*" -print0 | \
  while IFS= read -r -d '' f; do
    awk 'BEGIN{s=0} /^---$/{s++; next} s==1 && /^categories:/{p=1; next} s==1 && p && /^[a-zA-Z_]+:/{p=0} s==1 && p' "$f" | \
      rg -q "\\[\\[$CAT\\]\\]" && dirname "$f"
  done | sort | uniq -c | sort -rn
```

If 80%+ are in one folder — it's a mega-category candidate.

### B. Render-only edges (dashboards as topics)

Files have `topics: [[Dashboard - X]]` or similar, where the "topic" is a view-page that renders queries. The topic doesn't carry semantic meaning — it tells the file *where it gets displayed*, not *what it's about*.

**Signature:** A file named `Dashboard - X` (or similar view-page) appears as a topic on many other files. The dashboard itself contains Dataview/Bases queries, filtering by path or tag — not by topic.

**Why noise:** N files → 1 dashboard = N edges with zero navigational value. You don't jump from a contact to a dashboard to learn about the contact; you go the other way (dashboard lists contacts).

**Fix:** Strip the dashboard topic from those files. The dashboard's own queries (`dv.pages('"path/X"')`) keep working — they never used the topic-edge for filtering.

**Diagnostic:**
```bash
TARGET="Dashboard - X"
find . -name "*.md" -not -path "./.obsidian/*" -print0 | \
  while IFS= read -r -d '' f; do
    awk 'BEGIN{s=0} /^---$/{s++; next} s==1 && /^topics:/{p=1; next} s==1 && p && /^[a-zA-Z_]+:/{p=0} s==1 && p' "$f" | \
      rg -q "\\[\\[$TARGET\\]\\]" && echo "$f"
  done
```

### C. Log files with inline-field metadata

A single file holds bullets like `- [name:: X] [author:: [[Y]]] [series:: [[Z]]] → [[Linked]]`. Each bullet creates 2-4 wikilink edges. With 30+ bullets the log becomes a 50+-degree fake hub.

**Signature:** File is append-only, mostly bullets with `[key:: value]` inline-fields. Consumed as a dataset by Dataview queries elsewhere in the vault.

**Why noise:** This is structured data, not conceptual content. Edges represent *rows*, not *associations*.

**Fix:** Tag the file `#no-graph` (or `#log`) and exclude via filter. Dataview/Bases queries pulling its data still work — they read the file directly, not the graph.

**Diagnostic:**
```bash
# Files with high inline-field density
find . -name "*.md" -not -path "./.obsidian/*" -print0 | \
  while IFS= read -r -d '' f; do
    count=$(rg -c '\[[a-z_]+::' "$f" 2>/dev/null || echo 0)
    [ "$count" -gt 10 ] && echo "$count	$f"
  done | sort -rn
```

### D. False category (project name as category)

A file has `categories: [[Project Name]]`, where "Project Name" is a folder/epic, not an entity type. Categories should answer "what kind of thing is this", not "which project does this belong to".

**Signature:** Few files use this category (1-3). Category name matches a folder or epic.

**Why noise:** Creates a stray category-page node tied to those few files, masquerading as a real entity-class hub.

**Fix:** Remove the false category. If thematic association is needed, add a `topics:` entry instead.

**Diagnostic:**
```bash
# Categories with low usage — likely false or singletons
find . -name "*.md" -not -path "./.obsidian/*" -print0 | \
  while IFS= read -r -d '' f; do
    awk 'BEGIN{s=0} /^---$/{s++; next} s==1 && /^categories:/{p=1; next} s==1 && p && /^[a-zA-Z_]+:/{p=0} s==1 && p' "$f"
  done | rg -o '\[\[([^\]]+)\]\]' -r '$1' | sort | uniq -c | sort -n | head -20
```

### E. Broken placeholders (template leakage)

Wikilinks like `[[<placeholder>]]`, `[[{{var}}]]`, or `[[Title|alias]]\` (escaped pipe) appear in `topics:` or `categories:`. Created when a template was copied without substituting variables.

**Signature:** Wikilink content contains `<`, `>`, `{{`, `}}`, `\|`, or other template syntax. Often single-occurrence.

**Why noise:** Creates ghost or unresolved nodes, adding fake centrality.

**Fix:** Inspect each — usually edit the line out. Check carefully: a placeholder *inside* a code block is intentional documentation (`~~~markdown` example showing template format) — Obsidian doesn't create graph edges from code blocks. False alarm.

**Diagnostic:**
```bash
rg -l '\[\[<|\[\[\{\{|\\\|' --type md -g '!.obsidian/**' -g '!Templates/**'
```

Exclude `Templates/` since intentional placeholders live there.

### F. Heading-anchor topics (TOC in frontmatter)

A file has many `topics: - [[#Heading]]` entries — these are self-references to internal headings, written in `topics:` rather than the body.

**Signature:** `topics:` block has 10+ entries, all starting with `[[#`.

**Why "noise":** Cosmetic, not graph-breaking. Obsidian renders `[[#X]]` as a self-loop, not a separate node, so the graph isn't affected. But the entries inflate the file's apparent topic count and confuse diagnostics.

**Fix:** Move the TOC into the body (`## Navigation` block). Frontmatter is for metadata; in-document navigation belongs in the body.

**Diagnostic:**
```bash
find . -name "*.md" -not -path "./.obsidian/*" -print0 | \
  while IFS= read -r -d '' f; do
    awk 'BEGIN{s=0} /^---$/{s++; next} s==1 && /^topics:/{p=1; next} s==1 && p && /^[a-zA-Z_]+:/{p=0} s==1 && p' "$f" | \
      rg -q '\[\[#' && echo "$f"
  done
```

### G. Cross-domain meta-category bridge

A meta-category like `[[Projects]]` is on every project hub across unrelated life domains (work, cooking, hobbies, family). The single category page becomes a node that bridges domains visually.

**Signature:** A category page has many inbound edges from hub-files in different folders/domains. The graph shows seemingly unrelated clusters connected through this single node.

**Why noise (sometimes):** The meta-category is functional for navigation/dashboards but visually unifies things the user thinks of as separate.

**Fix — two options:**
- **Soft:** keep the category, hide its node from graph: `-file:"Projects"` in search filter.
- **Hard:** remove the meta-category from hub-files. Hubs stay visible via their own `#hub` tag or color group.

Choose soft if the category-page is used elsewhere (Bases view, navigation). Hard if the meta-category exists only because of a template default.

### H. Entity-name tags duplicating property fields

A tag value is a noun referring to an existing entity in the vault (an author, director, person, concept), and the same entity is already referenced via a frontmatter property field (e.g. `author: "[[J. R. R. Tolkien]]"`, `director: "[[Christopher Nolan]]"`).

**Signature:** Tags like `tolkien`, `nolan`, `tim-ferriss`, `paris` — short, lowercased, kebab-cased nouns matching existing file names. Look for `<tag-value>.md` (or close variants) in the vault.

**Why noise:** A flat tag doesn't create a graph edge. A property field with a `[[wikilink]]` already does. So the tag is duplicate metadata that pollutes frontmatter without adding graph signal. It also accumulates a tag taxonomy of one-off entity nouns, which makes search-by-tag less useful (the long tail floods the tag list).

**Fix — two cases:**
- **Property field already has the entity wikilink** → remove the tag (it's a pure duplicate).
- **No property field reference** → remove the tag and add `topics: [[Entity Name]]`. This creates the graph edge that was missing.

This is parallel to Pattern A (mega-category from folder-as-category): both patterns address frontmatter that uses one mechanism when another mechanism would carry more semantic weight.

**Diagnostic — find candidates:**
```bash
# List all tags by frequency. Entity-name tags will have noun-like values
# (author names, person names, place names) rather than category-like values
# (interview, homework, status/done).
find . -name "*.md" -not -path "./.obsidian/*" -print0 | \
  while IFS= read -r -d '' f; do
    awk 'BEGIN{s=0} /^---$/{s++; next} s==1 && /^tags:/{p=1; next} s==1 && p && /^[a-zA-Z_]+:/{p=0} s==1 && p' "$f"
  done | sed -e 's/^[ -]*//' -e 's/^\[//' -e 's/\]$//' -e 's/, /\n/g' | \
  sort | uniq -c | sort -rn | head -50
```

For each suspect tag value, check if a matching `*.md` file exists. If yes, it's a candidate for conversion.

**Diagnostic — for one specific tag:**
```bash
TAG="tolkien"
ENTITY="J. R. R. Tolkien"   # matching file basename
find . -name "*.md" -not -path "./.obsidian/*" -print0 | \
  while IFS= read -r -d '' f; do
    has_tag=$(awk 'BEGIN{s=0} /^---$/{s++; next} s==1' "$f" | rg -q "\b$TAG\b" && echo Y || echo N)
    has_wl=$(awk 'BEGIN{s=0} /^---$/{s++; next} s==1' "$f" | rg -q "\[\[$ENTITY\]\]" && echo Y || echo N)
    [ "$has_tag" = Y ] && echo "tag=$has_tag  wl=$has_wl  $f"
  done
```

Files with `tag=Y wl=Y` → tag is duplicate, just remove. Files with `tag=Y wl=N` → convert to topic.

**Caveat — preserve functional tags:** Some entity-name-like tags are intentionally used by Dataview/Bases queries for filtering. Before bulk-converting, search the vault for `WHERE contains(tags, "<tag>")` or `tags:contains("<tag>")` — if a query depends on the tag, removing it breaks the view. Confirm tag-by-tag before refactoring.

**Disambiguation:** A tag like `john` may map to multiple entities (`John Smith.md` vs `John Doe.md`). Resolve per-file by inspecting frontmatter (`author:`, `attendees:`, `interviewer:` fields), co-occurring tags, or body content. Don't bulk-rewrite ambiguous tags without classification.

**Caveat — exclude `Templates/` from bulk runs.** Templates may contain interpolated placeholders that look like entity tags but are actually template syntax. Common forms:

- Obsidian core templates: `{{date:MMMM}}`, `{{title}}` — replaced when a note is created from the template
- Templater plugin: `<% tp.date.now("MMMM") %>`, `<% tp.file.title %>`
- Others depending on plugins

A tag like `plan/{{date:MMMM}}` looks broken but is intentional — it expands to `plan/May`, `plan/June` etc. on note creation. Removing it from a template breaks all future notes generated from it.

Always exclude `Templates/` from any bulk script unless template hygiene is explicitly the goal:

```python
dirnames[:] = [d for d in dirnames if d not in (".obsidian", ".trash", ".claude", "Templates")]
```

## Mitigation patterns

These run **only after the user has approved a specific action** during step 5. They are reference recipes, not a default sequence — never run a mitigation just because a pattern was detected. The user picks which pain points to act on.

### Pre-flight checklist before `--apply`

Before running any bulk frontmatter rewrite, do these four checks. Skipping them is how silent damage happens (a real example: a cleanup script removed `plan/{{date:MMMM}}` from three template files because the script walked all folders by default; the placeholder is intentional Obsidian template syntax, and stripping it broke note creation).

1. **Run dry-run first.** The script must print, not write, by default. Apply only after the printed report looks right.
2. **Group affected files by top-level folder.** A one-liner on the dry-run output:
   ```bash
   python3 cleanup.py | rg '^\./[^/]+/' -o | sort | uniq -c | sort -rn
   ```
   If `Templates/`, `Attachments/`, or any vault internal folder appears — pause. Templates contain intentional placeholders that look like noise. Plugins may store config in folders you don't recognize.
3. **Inspect three random affected files inline.** Read the current frontmatter of three files the dry-run reports as modified. Confirm the planned change is correct for each. If even one looks wrong, the rule is wrong, not the file.
4. **Exclude Templates/ unless template hygiene is the goal.** This is the default for any bulk script — interpolated placeholders (`{{date:MMMM}}`, `<% tp.date.now %>`, etc.) look like broken refs but are required for new-note creation:

   ```python
   dirnames[:] = [d for d in dirnames if d not in (".obsidian", ".trash", ".claude", "Templates")]
   ```

Only after all four checks pass, run with `--apply`. If a problem surfaces post-apply, `git checkout -- <paths>` reverts cleanly because every change is an isolated frontmatter edit.

### Bulk strip a topic from frontmatter

When stripping a render-only topic from many files, use a Python script (more reliable than dozens of individual `Edit` calls). Skeleton:

```python
import os, sys
TARGET = "Dashboard - X"
DRY_RUN = "--apply" not in sys.argv
modified = removed_block = topics_removed = 0

for dirpath, dirnames, files in os.walk("."):
    dirnames[:] = [d for d in dirnames if d not in (".obsidian", ".trash", ".claude", "Templates")]
    for fn in files:
        if not fn.endswith(".md"): continue
        path = os.path.join(dirpath, fn)
        text = open(path, encoding="utf-8").read()
        if not text.startswith("---\n"): continue
        try: end = text.index("\n---\n", 4)
        except ValueError: continue
        fm, body = text[4:end], text[end+5:]
        lines = fm.split("\n")
        ts = next((i for i, ln in enumerate(lines) if ln == "topics:"), -1)
        if ts < 0: continue
        te = ts + 1
        while te < len(lines) and lines[te].startswith("  -"): te += 1
        items = lines[ts+1:te]
        new_items = [x for x in items if f"[[{TARGET}]]" not in x]
        if len(new_items) == len(items): continue
        new_lines = list(lines)
        if new_items:
            new_lines[ts:te] = ["topics:"] + new_items
        else:
            del new_lines[ts:te]
            removed_block += 1
        topics_removed += len(items) - len(new_items)
        modified += 1
        if not DRY_RUN:
            open(path, "w", encoding="utf-8").write("---\n" + "\n".join(new_lines) + "\n---\n" + body)

print(("DRY-RUN" if DRY_RUN else "APPLIED"),
      f"modified={modified} removed_block={removed_block} topics_removed={topics_removed}")
```

Always run with default (dry-run) first. Confirm the counts look right before adding `--apply`.

### Demote category to topic (mega-category fix)

Same shape as above — strip the category, append the same wikilink to topics. If the categories list becomes empty, drop the whole `categories:` block.

### Establish a tag taxonomy for hide

| Tag | Marks | Hidden via |
|---|---|---|
| `#dashboard` | Rendered view pages (kanban, metrics, charts) | `-tag:#dashboard` |
| `#no-graph` | Logs, configs, setup docs, infrastructure files (pure control tag) | `-tag:#no-graph` |
| `#archive` | Archived content kept for history | `-tag:#archive` |
| `#hub` | Project entry points (kept *visible*, often colored bright) | (no exclude) |

Filter:
```json
"search": "-tag:#dashboard -tag:#no-graph -tag:#archive"
```

This scales as the vault grows — a new dashboard just needs the tag, no graph-config edit.

## Decision matrix

When inspecting a high-degree node, ask:

| Question | Answer | Action |
|---|---|---|
| Is this concept the file is *about*? | Yes | Keep — semantic hub |
| Is this a view of other files? | Yes | `#dashboard`, exclude |
| Is this the file's storage location, not its topic? | Yes | Strip from `topics:`, keep file in folder |
| Does removing this edge lose meaning? | No | Strip |
| Would I navigate from here to there? | No | Strip |

## Caveats

- **Don't mistake hub-pages for noise.** A `Recipes.md` index file with 60 wikilinks to recipe entries is a healthy fan-pattern, not noise. Test: does the index help navigation, or just list files mechanically?
- **Don't strip topics that file owners actually use.** Confirm before bulk operations. Dry-run first is non-negotiable.
- **`Templates/` files often look broken** — placeholders are intentional. Exclude templates from audit unless template hygiene is the goal.
- **Code blocks may contain `[[wikilinks]]`** that look like real links but Obsidian doesn't render them as graph edges. False positives in counts; verify suspect files individually.
- **Inline-field counts can mislead.** A file with `[key:: value]` syntax is Dataview-readable metadata. If the file is meant as a dataset, the inline-field hub-degree is by design — solution is to hide via tag, not to strip fields.
- **Property-based searches use substring match.** `[\"categories\":\"Recipe\"]` matches both `Recipes` and `Recipe Drafts`. Verify category names are distinct strings before relying on property-search filters.
- **Obsidian rewrites `graph.json` when Graph view closes.** Always close the Graph tab (or quit Obsidian) before editing the file externally — otherwise edits are wiped.
