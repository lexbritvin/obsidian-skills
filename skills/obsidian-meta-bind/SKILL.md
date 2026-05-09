---
name: obsidian-meta-bind
description: Authoring buttons, input fields, view fields, and embeds for the Obsidian Meta Bind plugin. Use whenever the user asks to add an interactive control, button, input, viewfield, INPUT[...] / VIEW[...] / BUTTON[...] syntax, or a meta-bind code block. Covers schema, valid enum values, inline references, JS Engine integration, and the most common validation pitfalls (e.g. "Invalid input at 'style'", "Button ID not found", trailing-comma traps).
---

# Meta Bind — buttons, inputs, view fields

This skill covers the **Meta Bind** Obsidian plugin (id `obsidian-meta-bind-plugin`, by Moritz Jung). Authoring rules, full schemas, JS Engine integration, and the validation traps that cost hours when not known up front.

Docs root: <https://www.moritzjung.dev/obsidian-meta-bind-plugin-docs/>.

## When to use this skill

Reach for this skill on any of:

- User mentions Meta Bind, `INPUT[...]`, `VIEW[...]`, `BUTTON[...]`, ` ```meta-bind-button `, ` ```meta-bind ` code blocks.
- User wants to add an interactive widget into a note (toggle, slider, button, dropdown, date picker, etc.).
- User reports `META_BIND_ERROR`, `[MB_BUTTON]`, `Button ID not Found`, "validation failed", "Invalid input at 'style'", or any Meta Bind error overlay.
- User wants to bind metadata across files (`file^property`, `memory^...`, `globalMemory^...`).
- User needs to wire a button to an Obsidian command, JS code, file open, metadata mutation, or templater.
- User authors a tracker or dashboard with declarative INPUT fields tied to frontmatter.

## Prerequisites

Confirm before debugging:

1. **Meta Bind plugin installed and enabled.** Check `.obsidian/plugins/obsidian-meta-bind-plugin/manifest.json`.
2. **JS Engine plugin** installed and enabled if any button uses `action.type: inlineJS` or `js`. Without it, JS actions silently fail.
3. **Settings flag `enableJs: true`** in `.obsidian/plugins/obsidian-meta-bind-plugin/data.json`. JS-based buttons and JS view fields don't run otherwise.
4. **`excludedFolders`** in the same `data.json`: any path matching an entry is skipped by Meta Bind. Default contains `templates`. If a tracker file lives under `Templates/`, Meta Bind ignores it. Move the file or remove the exclusion.

## Button schema — every field

Source: <https://www.moritzjung.dev/obsidian-meta-bind-plugin-docs/guides/buttons/#button-configuration>.

| Field | Type | Required | Notes |
|---|---|---|---|
| `label` | string | **yes** | Display text on the button. |
| `style` | enum | **yes** | One of `default`, `primary`, `destructive`, `plain`. Validation fails if missing or out of enum — error reads `Invalid input at 'style'`. |
| `icon` | string | no | Lucide icon name. |
| `class` | string | no | Space-separated CSS classes. |
| `cssStyle` | string | no | Inline style rules (CSS text). |
| `backgroundImage` | string | no | URL or vault path. |
| `tooltip` | string | no | Hover text. Defaults to `label` when unset. |
| `id` | string | no | Used for `BUTTON[id]` inline references in the same note. |
| `hidden` | boolean | no | When `true` the code-block declaration is not rendered itself; only `BUTTON[id]` references render the button. |
| `action` | object | no* | Single action; mutually exclusive with `actions`. |
| `actions` | object[] | no* | Array of actions executed in sequence; mutually exclusive with `action`. |

*One of `action` or `actions` is required for the button to do anything.

### Minimal working button

```meta-bind-button
label: Run Script
style: primary
action:
  type: inlineJS
  code: console.log('clicked')
```

### Inline reference + hidden declaration

```markdown
Click here: `BUTTON[my-button-id]`

```meta-bind-button
label: Do thing
id: my-button-id
style: primary
hidden: true
action:
  type: command
  command: app:reload
```
```

The declaration is hidden; the inline `BUTTON[my-button-id]` renders the actual button at the cursor position. If `BUTTON[x]` shows `Button ID not Found`, the declaration with `id: x` either doesn't exist in the same file or **failed validation** and was dropped from the index — open the error overlay to see why.

## Button actions — types and schemas

Source: <https://www.moritzjung.dev/obsidian-meta-bind-plugin-docs/reference/buttonactions/...>. Action types confirmed in the plugin source: `command`, `open`, `input`, `sleep`, `inlineJS`, `js`, `templaterCreateNote`, plus metadata actions.

### `command` — run an Obsidian command

```yaml
action:
  type: command
  command: obsidian-meta-bind-plugin:open-faq
```

Find command IDs via the Meta Bind helper command "Select and copy command id".

### `open` — open a file or URL

```yaml
action:
  type: open
  link: "[[Some Note]]"     # or "https://example.com"
  newTab: false             # optional
```

### `inlineJS` — run JS code embedded in YAML

Requires JS Engine plugin + `enableJs: true` in Meta Bind settings.

```yaml
action:
  type: inlineJS
  code: |
    const file = app.workspace.getActiveFile();
    new Notice("hi from " + file.basename);
```

Inside the code, two extras are available: `context.buttonConfig` (the button's own config, read-only) and `context.buttonContext` (button metadata). Standard Obsidian globals — `app`, `Notice`, `window.moment`, etc. — all work.

### `js` — run a JS file from the vault

```yaml
action:
  type: js
  file: scripts/myAction.js
```

The JS file lives in the vault. Same JS Engine + `enableJs` requirement.

### `sleep` — pause between actions (only useful with `actions:`)

```yaml
action:
  type: sleep
  ms: 500
```

### `input` — type text at the cursor (in the editor)

```yaml
action:
  type: input
  str: "some text"
```

### `insertIntoNote` — insert text or Templater output at a line

```yaml
action:
  type: insertIntoNote
  line: "selfEnd + 1"     # absolute number or relative expression
  value: "Hello, world!"  # plain text — or a Templater template path
  templater: false        # set true if value is a template path
```

Relative-line expressions: `selfStart`, `selfEnd`, `selfStart - 1`, `selfEnd + 1` (relative to the button's own position). Absolute number also accepted.

### `replaceInNote` — replace a line range

```yaml
action:
  type: replaceInNote
  fromLine: 3
  toLine: 5
  replacement: "## New section\n\nfresh content"
  templater: false        # true → replacement is a template path
```

### `templaterCreateNote` — new note from a Templater template

```yaml
action:
  type: templaterCreateNote
  templateFile: "Templates/Lecture.md"
  folderPath: "Lectures"          # optional, default = vault root
  fileName: "New Lecture - RENAME ME"  # optional
  openNote: true                  # optional
  openIfAlreadyExists: false      # optional — opens existing instead of duplicate
```

### `createNote` — new empty note

```yaml
action:
  type: createNote
  fileName: "New Note"
  folderPath: "My Folder"     # optional
  openNote: false             # optional
  openIfAlreadyExists: false  # optional — appends running number on conflict
```

### Other action types (consult docs when needed)

- `setMetadata` / `updateMetadata` — write to frontmatter at a bind target.
- `regexp` — regex search-and-replace within the host note.
- `runTemplaterFile` — execute a Templater template as a script (no new note).

Reference URL pattern: `/reference/buttonactions/<typeName>/`.

### Multiple actions in one button

```yaml
actions:
  - type: command
    command: theme:use-light
  - type: sleep
    ms: 1000
  - type: command
    command: theme:use-dark
```

## Input fields — schema

Inline syntax: `` `INPUT[type(args):bindTarget]` ``. Arguments go inside `type(...)`. The bind target after `:` is optional but **strongly recommended** — without it, the input loses state on note reload.

Full type list:

| Type | Inline form | Common arguments |
|---|---|---|
| `toggle` | `INPUT[toggle:done]` | `onValue()`, `offValue()` |
| `slider` | `INPUT[slider(minValue(0), maxValue(10)):rating]` | `minValue()`, `maxValue()`, `stepSize()` |
| `text` | `INPUT[text(placeholder('name')):name]` | `placeholder()`, `defaultValue()` |
| `textArea` | `INPUT[textArea:notes]` | `placeholder()` |
| `number` | `INPUT[number(minValue(0)):count]` | `minValue()`, `maxValue()` |
| `date` | `INPUT[date:due]` | — |
| `time` | `INPUT[time:meet]` | — |
| `datePicker` | `INPUT[datePicker:deadline]` | — |
| `select` | `INPUT[select(option(a), option(b)):choice]` | `option()` |
| `multiSelect` | `INPUT[multiSelect(option(x), option(y)):tags]` | `option()` |
| `inlineSelect` | `INPUT[inlineSelect(option(yes), option(no)):flag]` | `option()` |
| `suggester` | `INPUT[suggester(option(A), option(B)):val]` | `option()` |
| `editor` | `INPUT[editor:body]` | — |
| `progressBar` | `INPUT[progressBar(minValue(0), maxValue(100)):pct]` | `minValue()`, `maxValue()` |
| `list` | `INPUT[list:items]` | — |
| `listSuggester` | `INPUT[listSuggester(option(x)):items]` | `option()` |
| `imageSuggester` | `INPUT[imageSuggester:img]` | — |
| `inlineList` | `INPUT[inlineList:keywords]` | — |
| `inlineListSuggester` | `INPUT[inlineListSuggester(option(t)):tags]` | `option()` |

`option()` two-arg form `option(value, displayName)` distinguishes stored value from rendered label.

To author a more complex input field, use a fenced code block:

````markdown
```meta-bind
INPUT[select(
  option(low, "Low priority"),
  option(med, "Medium"),
  option(high, "High priority"),
  class(my-css-class)
):priority]
```
````

### Input field arguments — full list

Arguments go inside `INPUT[type(arg1, arg2, ...):bindTarget]`. Multiple comma-separated.

| Argument | Applies to | Form | Example |
|---|---|---|---|
| `option(value)` / `option(value, displayName)` | select, multiSelect, suggester, listSuggester, inlineSelect, inlineListSuggester, imageSuggester | Two-form: with one arg, value = displayName. With two args, first = stored value, second = visible label. Type-coerced: `true`/`false` → bool, `null` → null, numeric → number. | `option(1, "1 Star")` |
| `optionQuery(query)` | suggester family | Dataview-like query that resolves to a list of options dynamically. | `optionQuery("#tag")` |
| `defaultValue(value)` | text, textArea, number, date, time | Pre-fills when bind target is empty. | `defaultValue("Untitled")` |
| `placeholder(text)` | text, textArea, number | Greyed-out hint inside an empty input. | `placeholder("Enter name")` |
| `minValue(n)`, `maxValue(n)`, `stepSize(n)` | slider, number, progressBar | Numeric bounds and step. | `minValue(0), maxValue(10), stepSize(0.5)` |
| `addLabels` | slider | Show numeric labels alongside the slider. | `addLabels` |
| `allowOther` | suggester family | Accept user input outside the option list. | `allowOther` |
| `class(className)` | all input field types | Add CSS class for custom styling. Multiple supported. | `class(my-css-class)` |
| `showcase` | all | Display the bind target name + value beside the input — useful for debugging. | `showcase` |
| `title(text)` | all | Override the visible label / accessibility title. | `title("Priority")` |
| `useLinks(true\|false)` | suggester variants | Whether selected values are stored as `[[wikilinks]]`. | `useLinks(true)` |
| `onValue(v)`, `offValue(v)` | toggle | Override what gets written for true/false (default: `true`/`false` booleans). | `onValue("done"), offValue("todo")` |

Class names with special chars must be quoted: `class("group-1")`.

For value strings with special chars (commas, parens), use single or double quotes inside the argument: `option("yes, please", "Yes")`.

## View fields — schema

Inline syntax: `` `VIEW[expression][type]` ``. Default `type` is `math`; can be omitted for math expressions.

Types: `math`, `text`, `image`, `link`, plus `jsViewField` for arbitrary JS-driven views.

Property references inside `expression` use `{propertyName}` and follow the same bind target rules as inputs.

```markdown
`VIEW[round(number({distance} km, miles), 2)]`           <!-- math (default) -->
`VIEW[**{title}**][text(renderMarkdown)]`                <!-- bold via text type -->
`VIEW[{cover}][image]`                                   <!-- image from frontmatter -->
`VIEW[{relatedNote}][link]`                              <!-- clickable link -->
```

## JS View Fields

`jsViewField` is a code block that lets arbitrary JS render a value, with reactive bind targets. It needs **JS Engine** + `enableJs: true`.

```meta-bind-js-view
bindTargets:
  distance: this#distanceKm
hidden: true
---
return Math.round(distance * 0.621371);
```

The runner injects each `bindTarget` as a top-level JS variable. The function returns the rendered value (string / number / DOM Node). Updates re-run when any binding changes.

Use cases: charts, computed labels, lookups in a contact list, anything beyond the math/text view fields.

## Meta Bind Embeds

A way to embed a note's interactive widgets *as if they belonged to the host file*. Standard `![[Note]]` keeps widgets bound to the source's frontmatter; meta-bind embeds re-bind to the **container** note.

Inline:

```
`META_BIND_EMBED[[Note A]]`
```

Block:

````
```meta-bind-embed[[Note A]]```
````

Use case: a "tracker template note" with INPUT/VIEW/BUTTON declarations, embedded into many host notes. Each host note gets its own bound state.

Caveat: deep nesting (embed-inside-embed) is limited; check current docs version.

## Settings (`data.json`) — the full set

| Key | Default | Effect |
|---|---|---|
| `devMode` | false | Dev-mode logs in console. |
| `ignoreCodeBlockRestrictions` | false | If true, Meta Bind also processes plain code blocks, not just its own. Risky. |
| `preferredDateFormat` | `YYYY-MM-DD` | Used by date inputs / view fields. |
| `firstWeekday` | `{index:1,name:"Monday",...}` | Calendar pickers. |
| `syncInterval` | 200 | ms between metadata sync passes. |
| `enableJs` | false | Toggles `inlineJS` / `js` actions and `jsViewField`. Required for any JS-driven element. |
| `viewFieldDisplayNullAsEmpty` | false | If true, missing values render as empty string instead of `null`. |
| `enableSyntaxHighlighting` | true | YAML highlighting in meta-bind code blocks. |
| `enableEditorRightClickMenu` | true | Right-click → "Insert input field/button". |
| `inputFieldTemplates` | [] | Reusable INPUT-field templates. Format: `{ id, declaration }`. Reference via `INPUT[id][overrides]`. |
| `buttonTemplates` | [] | Reusable button templates. Same fields as a button declaration, plus `id`. Reference via `` `BUTTON[id]` ``. |
| `excludedFolders` | `["templates"]` | Path-substring matches; Meta Bind skips files under those folders entirely. |

## ObsAPI — public methods

`const mb = app.plugins.getPlugin("obsidian-meta-bind-plugin").api;`

| Method | Purpose |
|---|---|
| `createField(filePath, opts)` | Generic field creator (input/view/jsView). |
| `createInputFieldMountable(filePath, opts)` | Build a mountable input field programmatically. |
| `createViewFieldMountable(filePath, opts)` | Build a mountable view field. |
| `createJsViewFieldMountable(filePath, opts)` | Build a mountable JS view field. |
| `createTableMountable(filePath, opts)` | Build a tracker/table mountable. |
| `createButtonMountable(filePath, opts)` | Build a mountable button (use this when rendering programmatic buttons from another plugin). |
| `createButtonGroupMountable(filePath, opts)` | Group of buttons. |
| `createEmbedMountable(filePath, opts)` | Programmatic meta-bind-embed. |
| `createExcludedMountable(filePath)` | The "excluded folder" placeholder UI. |
| `createInlineFieldFromString(str, ...)` | Parse a `INPUT[...]` / `VIEW[...]` / `BUTTON[...]` string. |
| `createInlineFieldOfTypeFromString(type, str, ...)` | Same but with explicit kind. |
| `wrapInMDRC(mountable, container, ctx)` | Lifecycle helper — handles mount/unmount via Obsidian's MarkdownRenderChild. **Always use this** when mounting a Meta Bind component manually. |
| `constructMDRCWidget(mountable, ...)` | Lower-level CodeMirror 6 widget constructor (live-preview path). |
| `createBindTarget(storageType, filePath, propertyPath)` | Programmatic bind target. |
| `parseBindTarget(str)` | Parse a bind-target string. |
| `createNotePosition(...)` | Position reference for `insertIntoNote` etc. |
| `createSignal(initial)` | Reactive signal — fire-and-subscribe. |
| `getMetadata(bindTarget)` | Read frontmatter / memory at a bind target. |
| `setMetadata(bindTarget, value)` | Write. |
| `updateMetadata(bindTarget, fn)` | Read-modify-write. |
| `subscribeToMetadata(bindTarget, fn)` | React to changes. |
| `reactiveMetadata(bindTarget)` | Wrap a bind target in a `Signal` for reactive UIs. |

There is **no public `setButtonTemplate` / `registerButtonTemplate`**. To populate templates programmatically, you must reach into `mb.api.mb.buttonManager.buttonTemplates` (internal). See "zero data.json" pattern above.

## Bind targets — full syntax

`storageType^storagePath#property`. Both `storageType^` and `storagePath#` are optional.

| Storage type | Notes |
|---|---|
| `frontmatter` (default) | YAML at the top of a file. |
| `memory` | Per-file ephemeral; lost when no file references it for a while or on Obsidian restart. |
| `globalMemory` | Vault-wide ephemeral, non-persisted. |
| `scope` | Extends another bind target without explicit storage path. |

Examples:

```
INPUT[toggle:completed]                    # local frontmatter
INPUT[toggle:Task A#completed]             # frontmatter in another file
INPUT[toggle:path/to/Task A#completed]     # frontmatter in subfolder
INPUT[text:memory^Task A#draft]            # ephemeral memory keyed to a file
INPUT[text:globalMemory^#counter]          # vault-wide ephemeral
INPUT[toggle:this.is.nested]               # nested via dot
INPUT[toggle:["with spaces"]]              # bracketed for special chars
```

## Common validation traps (the painful ones)

1. **`style` is required.** Missing or out-of-enum value triggers `Invalid input at 'style'`. Always set one of `default | primary | destructive | plain`.

2. **JS Engine plugin missing.** Buttons with `inlineJS` / `js` action will fail to run (often silently — check developer console). Install JS Engine and toggle `enableJs: true` in Meta Bind settings.

3. **`excludedFolders`** in `data.json` (default `["templates"]`) makes Meta Bind skip any file under those folders entirely — buttons there won't render. Match is path-substring against the lowercased file path.

4. **Trailing commas in inlineJS code blocks.** Object/array literals with trailing commas can fail validation in Meta Bind 1.4.x — strip them. Convert
   ```js
   tk.fn({ a: 1, b: 2, })
   ```
   to
   ```js
   tk.fn({ a: 1, b: 2 })
   ```

5. **Multi-line object literals in `inlineJS`.** Less reliable than inline form — when in doubt, write the call on a single line, or hoist the object into a variable on its own line:
   ```yaml
   action:
     type: inlineJS
     code: |
       const opts = { foo: "x", bar: "y" };
       await someFn(opts);
   ```

6. **`Button ID not Found`** on `BUTTON[id]` inline reference. Causes:
   - declaration block has a `style` / schema validation error — it was dropped from the index;
   - `id:` in the declaration doesn't match the inline reference;
   - declaration is in a different file (Meta Bind only resolves IDs in the same note);
   - file lives under an `excludedFolders` entry;
   - **template registered programmatically, but rendering ran first** — see trap #12 (race on cold start).

7. **YAML quoting around emojis and Unicode.** Emojis like 📂 in `label:` are fine when wrapped in `"..."`. Bare unquoted emoji at the start of a value can confuse the YAML parser; quote when in doubt.

8. **`hidden: true` is not a substitute for action.** Hidden declarations still need a valid `action` or `actions` — they're rendered through the inline reference, not skipped.

9. **`enableJs: true` flips a permission flag** but doesn't re-load currently rendered notes — toggling the flag, or reloading Obsidian (Cmd+P → Reload app without saving), is sometimes needed.

10. **Inline INPUT/VIEW in callout/blockquote.** Inline fields render normally, but watch for Markdown reader confusion when nested inside callouts — fenced `meta-bind` code blocks are safer for complex cases.

11. **`button with id "X" already exists in the button templates`.** Triggered when a `meta-bind-button` declaration in a file shares its `id` with a template already in `buttonManager.buttonTemplates` (whether from `data.json` or programmatic registration). Meta Bind's `addButton` is **not** idempotent — it throws on duplicate `id`. The error stack shows `Tn.addButton → pY.onMount` (component mount). Fix: pick one source of truth — keep the declaration **or** the template, never both. When migrating to programmatic registration, scrub all in-file `meta-bind-button` declarations of those ids and replace with bare `BUTTON[id]` references.

12. **`Button ID not Found` on first cold open, works on second.** Race between Meta Bind's first render pass and your plugin's programmatic registration. Meta Bind renders already-open files before registration runs → `BUTTON[id]` resolves to nothing → failed render is cached in the markdown view. Fix: use the full three-step lifecycle in the "zero data.json" pattern below — sync register in `onload`, re-register on `onLayoutReady`, cleanup on `onunload`. The sync `onload` step is what prevents the bug.

## Centralizing buttons via `buttonTemplates`

Inline references `BUTTON[id]` look up the button **first in `buttonTemplates`** (Meta Bind plugin settings, in `.obsidian/plugins/obsidian-meta-bind-plugin/data.json`), and only then in local declarations inside the same note. Source: `getButton(file, id) { if (this.buttonTemplates.has(id)) return this.buttonTemplates.get(id); ... }`.

**Use this when:** the same button is referenced from many notes, or you want notes to be free of YAML clutter.

`data.json` shape:

```json
{
  "buttonTemplates": [
    {
      "id": "open-notes",
      "label": "📂 Open notes",
      "style": "primary",
      "hidden": true,
      "action": {
        "type": "inlineJS",
        "code": "const target = app.vault.getAbstractFileByPath('Notes.md');\nawait app.workspace.getLeaf(false).openFile(target);\n"
      }
    }
  ]
}
```

After adding templates, in any note you write only:

```markdown
`BUTTON[open-notes]`
```

No `meta-bind-button` block needed.

**Versioning:** if the vault gitignores `.obsidian/plugins/*/data.json`, add an explicit exception for Meta Bind so templates travel with the vault:

```
.obsidian/plugins/*/data.json
!.obsidian/plugins/obsidian-meta-bind-plugin/data.json
```

**Editing templates:** Settings → Meta Bind → Button Templates in Obsidian, or edit `data.json` directly and reload the plugin. For bulk authoring, a script that writes to `buttonTemplates` is faster than the UI.

### Pattern: thin templates + logic in another plugin

When the vault has a helper plugin, the cleanest split is:

- **`data.json` `buttonTemplates`** holds *presentation* (label, style, icon, hidden flag) and a one-liner action that delegates to the helper plugin:
  ```json
  "action": {
    "type": "inlineJS",
    "code": "await app.plugins.plugins['helper-plugin'].api.runButton('my-button');\n"
  }
  ```
- **Helper plugin** holds the *registry* of what each button does — a map of `{ id → config }` keyed by button id, where each config picks the underlying primitive (open file, write a bullet, open a modal, etc.).
- A single dispatch function `runButton(id)` in the helper plugin reads the registry and calls the right primitive.

This keeps `data.json` minimal and uniform (every button is one line), keeps logic in normal JS files under git with refactor-friendly tooling, and survives Meta Bind upgrades without churn.

### Pattern: zero data.json — programmatic registration from helper plugin

Goal: eliminate `buttonTemplates` from `data.json` entirely. The helper plugin owns one in-code map of `{ id → presentation }` and pushes those into Meta Bind's `buttonManager.buttonTemplates` Map at startup.

Lifecycle has three steps; each lines up with a real failure mode. Use this exact shape.

| Step | When | What it does | Why it's needed |
|---|---|---|---|
| 1 | `onload` (sync) | Register templates | Beats Meta Bind to its first render of already-open files. Without this, `BUTTON[id]` shows "id not found" until the file is reopened. |
| 2 | `onLayoutReady` | Register again + rerender open reading-mode views | Covers the case where Meta Bind wasn't loaded yet during step 1 (rare load-order edge case); `previewMode.rerender(true)` clears any cached placeholder. |
| 3 | `onunload` | Delete our ids from the Map | Hot Reload of the helper plugin would otherwise leave stale templates that throw "already exists" on next register. |

```js
const MB_PLUGIN_ID = "obsidian-meta-bind-plugin";

class HelperPlugin extends Plugin {
  async onload() {
    this.api = { /* … */ };

    this.tryRegisterButtonTemplates();
    this.app.workspace.onLayoutReady(() => {
      this.tryRegisterButtonTemplates();
      this.refreshOpenMarkdownViews();
    });
  }

  tryRegisterButtonTemplates() {
    try { this.registerButtonTemplates(); }
    catch (e) { console.warn("[helper] meta-bind register failed:", e); }
  }

  registerButtonTemplates() {
    const map = this.app.plugins.getPlugin(MB_PLUGIN_ID)
      ?.api?.mb?.buttonManager?.buttonTemplates;  // INTERNAL API
    if (!map?.set) return;                        // Meta Bind not ready yet
    for (const id of Object.keys(MY_BUTTONS)) {
      map.set(id, {                               // .set replaces by key, idempotent
        id,
        label: MY_BUTTONS[id].label,
        style: MY_BUTTONS[id].style ?? "default",
        hidden: true,
        action: {
          type: "inlineJS",
          code: `await app.plugins.plugins['my-plugin'].api.fn('${id}');\n`,
        },
      });
    }
  }

  refreshOpenMarkdownViews() {
    this.app.workspace.iterateAllLeaves((leaf) => {
      const view = leaf?.view;
      if (view?.getViewType?.() !== "markdown") return;
      try { view.previewMode?.rerender?.(true); } catch (_) {}
    });
  }

  onunload() {
    try {
      const map = this.app.plugins.getPlugin(MB_PLUGIN_ID)
        ?.api?.mb?.buttonManager?.buttonTemplates;
      if (map) for (const id of Object.keys(MY_BUTTONS)) map.delete(id);
    } catch (_) { /* best effort */ }
    delete this.api;
  }
}
```

Caveats:

- `mb.api.mb.buttonManager.buttonTemplates` is **internal**. Subject to break on Meta Bind upgrades; if a public `setButtonTemplate` / `registerButtonTemplate` ships, switch to that.
- If the user opens Settings → Meta Bind → Button Templates and saves, Meta Bind re-runs `loadTemplates()` and wipes programmatic entries. Subscribe to settings-change events and re-register, or document the limitation.
- `data.json buttonTemplates: []` stays empty under git. Keep the gitignore exception (`!.obsidian/plugins/obsidian-meta-bind-plugin/data.json`) so the empty-array state travels with the vault.

## Embedding inline JS that calls plugin APIs

When a button calls a custom plugin's API (e.g. `app.plugins.plugins["my-plugin"].api.x.y(...)`), bear in mind:

- The plugin must be loaded; if it crashed during onload, the API is `undefined` and clicks no-op or throw `Cannot read properties of undefined`.
- Wrap with optional chaining for graceful failure: `app.plugins.plugins["x"]?.api?.fn?.()`.

## Working patterns

### Append a dated bullet to a section in the active file

Plain-JS write (no helper plugin) — useful when you don't want a registry but need a quick logger.

```meta-bind-button
label: "✓ Log today"
id: log-today
hidden: true
style: primary
action:
  type: inlineJS
  code: |
    const file = app.workspace.getActiveFile();
    const today = window.moment().format("YYYY-MM-DD");
    const content = await app.vault.read(file);
    await app.vault.modify(file, content + `\n- ${today}`);
    new Notice("logged " + today);
```

For anything more structured (sections, dedupe, sorted insertion, multi-field input) use a helper plugin's primitives — see the modal pattern below.

### Open or create a file via plugin helper

```meta-bind-button
label: "📂 Open monthly log"
id: open-monthly
hidden: true
style: default
action:
  type: inlineJS
  code: |
    await app.plugins.plugins["helper-plugin"].api.openMonthly({ folder: "Activity", templatePath: "Templates/Monthly.md" });
```

### Modal-based event capture (avoiding frontmatter buffers)

```meta-bind-button
label: "🌸 Add event"
id: event-add
hidden: true
style: default
action:
  type: inlineJS
  code: |
    const helper = app.plugins.plugins["helper-plugin"].api;
    await helper.recordWithModal({
      file: app.workspace.getActiveFile(),
      section: "## Events",
      title: "Add event",
      successMessage: "Recorded",
      fixedFields: { type: "event" },
      fields: [
        { key: "item", label: "Item", type: "text", required: true },
        { key: "cost", label: "Cost", type: "number" }
      ]
    });
```

## Quick checklist before shipping a button

- [ ] `label`, `style` set (style ∈ enum).
- [ ] `id` matches the inline `BUTTON[id]` reference exactly.
- [ ] `hidden: true` if you only want the inline-rendered button.
- [ ] `action.type` is one of the supported types.
- [ ] If `inlineJS` — JS Engine plugin is enabled, `enableJs: true`, no trailing commas, single-line object literal preferred.
- [ ] File is **not** under any `excludedFolders` path.
- [ ] If the action calls a custom plugin, that plugin's `api` is reachable and the namespace function exists.
- [ ] `Cmd+P → Reload app without saving` after schema or plugin changes.

## Proactive use cases — when to suggest Meta Bind

Watch for these user phrasings or contexts and propose Meta Bind solutions:

| User signal | Suggest |
|---|---|
| "I want a button to do X" / "click to record/append" | Meta Bind button with appropriate action type. |
| "I want to track {state} in this note" / status / completion | INPUT[toggle] / INPUT[select] bound to frontmatter. |
| "show {value} from frontmatter" / "computed value" | VIEW[…] math or text field. |
| "rate / score 1–5 / 1–10 in note" | INPUT[slider(addLabels, minValue(N), maxValue(M)):rating]. |
| "pick from a list of notes/tags" | INPUT[suggester(optionQuery("#tag")):field]. |
| "checklist for daily/weekly habit" | INPUT[multiSelect] or several INPUT[toggle]. |
| "this template should have working widgets in every copy" | Plain INPUT/VIEW in the template — they bind to each instance's frontmatter automatically. |
| "I want a single source widget reused across notes" | meta-bind-embed (rebinds widgets to host frontmatter) or buttonTemplates / inputFieldTemplates with reference. |
| "create a new note from this button" | createNote or templaterCreateNote action. |
| "insert text below the button" | insertIntoNote with `selfEnd + 1`. |
| "modify a frontmatter field on click" | setMetadata / updateMetadata action. |
| "memory state that survives page reload but not Obsidian restart" | bind target with `memory^` storage type. |
| "shared counter / state across all notes" | bind target with `globalMemory^` storage type. |
| "computed columns / dashboard tied to frontmatter" | createTableMountable via API, or VIEW[] fields with bind targets. |
| "interactive chart / chart that updates with note property" | jsViewField (returns a DOM node from JS). |

When suggesting, show the **inline syntax** first (one line) — it's cheap to drop into a note. Move to the code-block form when the user wants multiple arguments or class styling.

## Diagnostics — fast bisect when a button fails

1. Drop a clone of any known-working button into the broken file with a fresh `id` — rules out file-level / folder-level exclusion (e.g. `excludedFolders`).
2. If the clone works but the original doesn't, simplify the original's body to a single `new Notice("test")` — rules out JS-side bugs.
3. If still broken, remove fields one by one (`icon`, `class`, `cssStyle`, …) — Meta Bind's zod errors name field paths but occasionally mislabel; bisect is faster than introspection.
4. Open the Meta Bind error overlay (the rendered card on the broken button) — the structured zod path almost always names the offending field.
5. For "Button ID not Found" specifically: open the same file in a second tab to see if it works on a fresh render — distinguishes a cached failed render (race, see trap #12) from a missing template.
