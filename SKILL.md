---
name: obsidian-plugin-lint
description: 'Use when asked to lint, check, or review an Obsidian plugin''s CSS or code for style violations. Triggers on "lint", "检查代码", "check styles", before committing/releasing a plugin, or when seeing submission bot warnings: "Avoid !important", "Use the full 6-digit hex format", "Unexpected duplicate", "Unexpected browser feature", "Unexpected unknown type selector", "Unsafe assignment to innerHTML", "Sets styles directly instead of using CSS classes", "Unsafe call of an error or any typed value", "Unsafe member access on an error or any typed value", "Unsafe assignment of an error or any typed value", "Passes unsafe values into typed parameters", "This assertion is unnecessary since the receiver accepts the original type of the expression", "This assertion is unnecessary since it does not change the type of the expression", "overrides all other types in this union type", "display is deprecated", "Use getSettingDefinitions instead", "does not implement getSettingDefinitions", "will not appear in Obsidian''s settings search", "Creating and attaching "style" elements is not allowed", "Uses Obsidian APIs newer than the declared minAppVersion", "Avoid unnecessary logging to console", "Obsidian''s configuration folder is not necessarily .obsidian", "Use Vault#configDir to get the current value", "is defined but never used", "Use ''FileManager.trashFile()'' instead of ''Vault.trash()''", "setWarning is deprecated", "builtin-modules" should be replaced with an alternative package". Also handles the "/obsidian-plugin-publish [version]" release workflow — commit outstanding work, generate version.md from git history since the last release tag, bump manifest.json/package.json, tag, and push to trigger the release pipeline. Triggers on "/obsidian-plugin-publish", "发布插件", "打个 release", "bump version and tag".'
model: haiku
---

# Obsidian Plugin Lint

## Overview

Static analysis pass for Obsidian plugin code. Covers both official Obsidian ESLint rules and general CSS best practices. Run before every release.

## Rules

### Official Obsidian ESLint Plugin Rules (`obsidianmd/eslint-plugin`)

Source: https://github.com/obsidianmd/eslint-plugin

| Rule | What to check | Fix |
|------|---------------|-----|
| `no-static-styles-assignment` | `el.style.color = ...`, `el.style.cssText = ...`, `el.style.display = ...` (any inline style via JS) | CSS class: `el.addClass('my-class')`. For dynamic values: `setCssProps(el, { '--my-var': value })` or `setCssStyles(el, { display: value })` |
| `no-unsupported-api` | An API whose current signature is newer than `minAppVersion` (e.g. `workspace.revealLeaf` returning `Promise<void>` requires 1.7.2) | Bump `manifest.json` `minAppVersion` to the version that introduced the API; see section below |
| `obsidianmd/commands` (no plugin name) | `addCommand({ name })` repeats the plugin name (`"Open Day Echo timeline"` for plugin "Day Echo") | Drop the plugin name — Obsidian already prefixes it. `name: "Open timeline"` |
| Sentence case | Button labels, settings headings, command names must use sentence case | Capitalize only first word and proper nouns |
| No default hotkeys | `addCommand()` must not set `hotkeys: [...]` | Remove default hotkeys; let user configure |
| No `innerHTML` | Any `el.innerHTML = ...` assignment (not just user input) | Use `el.createEl()`, `el.createDiv()`, `el.setText()` — innerHTML is unsafe regardless of source |
| Use `this.app` | `window.app` or global `app` | Replace with `this.app` |
| `normalizePath()` | User-defined paths passed to Vault API without normalizing | Wrap with `normalizePath()` from `obsidian` |
| No console logging | `console.log`/`console.info` left in for routine, non-error events | Remove it; keep `console.error`/`console.warn` only for failures a bug report would need to diagnose |
| Don't hardcode `.obsidian` | A literal `".obsidian"` path segment building a config-folder path | Use `Vault#configDir` (`this.app.vault.configDir`) — the user can rename this folder |

### CSS Best Practices (official docs recommendation, not enforced by eslint-plugin)

| Practice | Violation | Fix |
|----------|-----------|-----|
| Use CSS variables | Hardcoded colors/sizes (`color: white`, `font-size: 12px`) | Use Obsidian variables: `var(--text-normal)`, `var(--font-ui-small)` |
| Avoid `!important` | Any `!important` declaration | Add `.workspace` parent selector to raise specificity |
| Full 6-digit hex | Shorthand hex `#fff`, `#aaa`, `#ccc`, `#888` | Expand to 6 digits: `#ffffff`, `#aaaaaa`, `#cccccc`, `#888888` |
| No duplicate properties | Same property declared twice in one rule block (e.g. `position: fixed` repeated) | Remove the duplicate; keep the intended declaration |
| Avoid `:has()` | Any `:has()` selector — broad selector invalidation hurts performance | Structural: target the matched element's own class. State-based: JS toggles a marker class, key CSS off it. Content-based (nothing in the note to target): mark the rendered DOM in both views, key CSS off the mark |
| Browser feature compatibility | Properties only partially supported by Obsidian's Electron/Chromium version (`clip-path`, `column-count`, `box-decoration-break`, newer `text-decoration-*` sub-properties) | Add vendor prefixes or avoid the feature; see section below |
| No unknown type selectors | Non-standard HTML element names in selectors (`webview`, `mjx-container`) | Use class selectors instead, or acknowledge as intentional third-party elements |

## Checking CSS for `!important`

**Submission bot warning:** `Avoid !important — override styles by increasing selector specificity or using CSS variables instead.`

```bash
grep -n '!important' styles.css
```

For each hit:

1. **First compute specificity instead of guessing** — before adding a `.workspace` prefix, check whether the existing selector *already* outranks every competing Obsidian rule on that property. Count (id, class, type) per selector (`:is()`/`:not()` contribute the specificity of their most specific argument, not an extra level) and compare against the native rules in `app.css` touching the same property/element. If the plugin's selector is already higher, `!important` is simply unused weight — drop it with no other change.
2. If it is *not* already higher, add `.workspace` as parent (or another real ancestor class) — outranks Obsidian's base rules (which don't use `!important` on these properties)
3. If Obsidian itself uses `!important` on the same property, keep it and document why

**Worked example (obsidian-minimalism-ui, 2026-11-06)** — `!important` added defensively in a ribbon-hiding fix, flagged by the submission bot:

```css
/* Before */
body.minimalism-ui-mac-sidebar .workspace-ribbon.side-dock-ribbon:is(.mod-left, .mod-primary) {
	display: none !important;
}
```

Competing native rules in `app.css`:

| Selector | Specificity (id, class, type) |
|---|---|
| `.workspace-ribbon { display: flex; }` | (0,1,0) |
| `body:not(.show-ribbon) .workspace-ribbon { display: none; }` | (0,2,1) |
| Plugin's selector (4 classes incl. `:is()`, + `body`) | **(0,4,1)** |

The plugin's selector already outranks both native rules without `!important` — neither native rule uses `!important` either, so specificity alone decides it regardless of which state (`show-ribbon` present or not) the body is in:

```css
/* After — same selector, just drop !important */
body.minimalism-ui-mac-sidebar .workspace-ribbon.side-dock-ribbon:is(.mod-left, .mod-primary) {
	display: none;
}
```

Only reach for `.workspace`-prefixing or keeping `!important` when the specificity table actually shows the plugin's selector losing.

**Worked example (obsidian-better-link-display, 2026-11-11)** — overriding a `<button>` element's background without `!important`. A hand-rolled segmented control styled a `<button class="…-radio">`; Obsidian's own button rules carry a type selector (`button`) plus theme classes, which outranks a bare class selector:

```css
/* Before — loses to Obsidian's button background, so !important papers over it */
.my-plugin-radio {
	background: transparent !important;
}
.my-plugin-radio:hover {
	background: transparent !important;
}

/* After — match the same shape: ancestor class + `button` type selector, and
   spell the :hover state out explicitly rather than relying on it to inherit
   from the base rule — Obsidian's own button:hover rule is also more specific
   than a lone class, so the override needs to win at that state too */
.my-plugin-panel button.my-plugin-radio {
	background: transparent;
}
.my-plugin-panel button.my-plugin-radio:hover {
	background: transparent;
}
```

Adding the real parent class (not `.workspace`, which wouldn't exist over a plugin's own panel) and the `button` type selector raises the selector's specificity past Obsidian's, with no `!important` needed.

Applies to all properties (`display`, `pointer-events`, `overflow`, `overflow-y`, etc.):

```css
/* Before */
.internal-embed.bases-toolbar-hidden .bases-toolbar { display: none !important; }
.internal-embed.bases-toolbar-hidden .bases-thead   { pointer-events: none !important; }
.internal-embed.bases-toolbar-hidden                { overflow: visible !important; }
.bases-lock-container .bases-view                   { overflow-y: hidden !important; }

/* After — add .workspace prefix, drop !important */
.workspace .internal-embed.bases-toolbar-hidden .bases-toolbar { display: none; }
.workspace .internal-embed.bases-toolbar-hidden .bases-thead   { pointer-events: none; }
.workspace .internal-embed.bases-toolbar-hidden                { overflow: visible; }
.workspace .bases-lock-container .bases-view                   { overflow-y: hidden; }
```

## Checking CSS for short hex codes

**Submission bot warning:** `Use the full 6-digit hex format for consistency.`

```bash
grep -nP '#[0-9a-fA-F]{3}\b' styles.css
```

Expand every 3-digit shorthand to 6 digits by doubling each nibble:

```css
/* Before */
color: #fff;
color: #aaa;
border: 1px solid #ccc;

/* After */
color: #ffffff;
color: #aaaaaa;
border: 1px solid #cccccc;
```

## Checking CSS for duplicate properties

**Submission bot warning:** `Unexpected duplicate "<property>".`

```bash
# Find property names that appear more than once in the same rule block
awk '/\{/{block=""} {block=block"\n"$0} /\}/{
  n=split(block,lines,"\n")
  for(i=1;i<=n;i++) {
    match(lines[i],/^[[:space:]]*([a-z-]+)[[:space:]]*:/,m)
    if(m[1]) count[m[1]]++
  }
  for(k in count) if(count[k]>1) print FILENAME ":" NR " duplicate: " k
  delete count
}' styles.css
```

Or just grep for a specific suspected property:

```bash
grep -n 'position:' styles.css
```

Remove the earlier (usually leftover) duplicate and keep the final intended declaration.

## Checking CSS for `:has()`

**Submission bot warning:** `Avoid :has — it can cause significant performance issues due to broad selector invalidation.`

`:has()` forces the browser to re-test the selector whenever anything inside the subject could change. On `body:has(...)` that means *every* DOM/state change in the document re-evaluates the rule. Obsidian flags it on submission.

```bash
grep -n ':has(' styles.css theme/*.css
```

Three patterns, three different fixes:

### 1. Structural — `:has(> .child)` selects a parent by its child

When `:has` only encodes DOM structure (no `:hover` / `:focus` / runtime state), the matched element is fixed — just target that element's own class directly.

```css
/* Before — select the live-preview table widget by its child */
body.note-style .cm-editor :has(> .table-wrapper) { overflow: visible; }

/* After — the parent is always .cm-table-widget; target it directly (same specificity) */
body.note-style .cm-editor .cm-table-widget { overflow: visible; }
```

Inspect the DOM once to learn the real parent class, then hardcode it.

### 2. State — `ancestor:has(descendant:state)` reacts to runtime state

When the selector depends on a descendant's live state (`:focus-within`, `:checked`, `.is-active`) **and** the element you want to style is not itself a descendant of the stateful node (classic case: a popover Obsidian `appendChild`s to `document.body`), pure CSS has no other handle. Replace `:has` with a tiny JS `Feature` that toggles a marker class on the ancestor, then key CSS off the marker.

```css
/* Before — restyle a body-level dropdown only while a sidebar field is focused */
body.mac-sidebar:has(.mod-left-split .metadata-property-value:focus-within) .suggestion-container { ... }

/* After — JS toggles .minimalism-ui-sidebar-prop-focus on body */
body.mac-sidebar.minimalism-ui-sidebar-prop-focus .suggestion-container { ... }
```

```typescript
// One closest() per focus change — far cheaper than :has re-invalidation.
// Use activeDocument / activeWindow (popout-window safe), not bare document / setTimeout.
const SEL = '.workspace-split.mod-left-split .metadata-property-value';
const sync = () => activeDocument.body.classList.toggle(
    'minimalism-ui-sidebar-prop-focus',
    !!activeDocument.activeElement?.closest(SEL),
);
activeDocument.addEventListener('focusin', sync);
// focusout fires before focus settles → re-check next tick
activeDocument.addEventListener('focusout', () => activeWindow.setTimeout(sync, 0));
```

Implement as a `Feature` (apply/remove) so listeners are torn down on unload. For non-focus state (class/attribute toggled by another plugin), drive the same marker-class toggle from a `MutationObserver` instead of focus events.

### 3. Content — `:has()` identifies *what the user wrote*, and there is no class to target

The hardest case, and the one that looks unfixable at first: `:has()` isn't encoding structure or runtime state, it's asking "does this element contain the thing my plugin produced?" Typical of plugins whose output is **plain Markdown with no marker in the note** (a deliberate invariant: the note must keep rendering with the plugin uninstalled), so the only signal is a content feature of the rendered DOM — an inlined `data:` image, a specific `href`, an emoji.

```css
/* Before — obsidian-better-link-display, 17 warnings, 2026-08-15 */
a.external-link:has(> img[src^="data:image/"]) { color: var(--text-muted); }
.cm-editor .cm-line:has(img[src^="data:image/"]) .cm-link { color: var(--text-muted); }
```

Neither fix above applies: the parent has no class of its own (#1), and nothing toggles at runtime (#2). **The fix is to do the recognition once in JS, on the rendered DOM, and leave a marker class behind for CSS** — the note still gets nothing written into it, because the marks only ever exist on rendered elements.

```css
/* After — every rule keys off a class the plugin put there */
a.external-link.better-link-display-bookmark { color: var(--text-muted); }
.cm-editor .cm-line.better-link-display-line .cm-link { color: var(--text-muted); }
```

**It has to be done for each view separately — Reading view and Live Preview render the same Markdown into different DOM** (which is exactly why these rules came in mirrored pairs to begin with), and inside Live Preview the mechanism differs *again* depending on who owns the element:

```typescript
// main.ts
this.registerMarkdownPostProcessor((el) => markReadingViewBookmarks(el));   // Reading view
this.registerEditorExtension(bookmarkMarkers);                              // Live Preview
```

**① Reading view — plain DOM marking in a Markdown post-processor.** Nothing else owns that DOM, and each block is passed in as it is rendered:

```typescript
export function markReadingViewBookmarks(el: Element): void {
    for (const icon of Array.from(el.querySelectorAll(ICON_SELECTOR)))
        icon.closest("a.external-link")?.addClass(BOOKMARK_CLASS);
}
```

**⚠️ ② Live Preview, `.cm-line` — you cannot mark it in the DOM. CodeMirror will strip the class.** This is the trap that makes the naive "just `querySelectorAll` the editor and `addClass`" version look like it works and then flicker. CodeMirror's `DOMObserver` observes the content DOM with `attributes: true, subtree: true`; a foreign class on a `.cm-line` produces a mutation record, `readMutation()` finds the `LineView` (which, unlike a widget, does **not** ignore mutations), and flags it `AttrsDirty` — and `LineView.sync()` then does exactly this:

```javascript
// node_modules/@codemirror/view/dist/index.js — LineView.sync()
else if (this.flags & 4 /* ViewFlag.AttrsDirty */) {
    clearAttributes(this.dom);
    this.dom.className = "cm-line";     // ← your class, gone
```

So the class survives until the next sync of that line and then vanishes, with no update to re-add it. **Use a `Decoration.line` instead** — the supported way to put a class on a line, and a bonus: it's computed from the *document text*, so it doesn't depend on a render having happened yet.

```typescript
const lineMark = Decoration.line({ class: LINE_CLASS });

function bookmarkLines(view: EditorView): DecorationSet {
    const marks: Range<Decoration>[] = [];
    const { doc, selection } = view.state;
    let lastMarked = -1;
    for (const { from, to } of view.visibleRanges) {
        for (let pos = from; pos <= to; ) {
            const line = doc.lineAt(pos);
            pos = line.to + 1;
            // Two visible ranges can meet inside one line — never mark twice at one pos.
            if (line.from <= lastMarked || !line.text.includes(ICON_MARKUP)) continue;
            // Live Preview shows the raw Markdown on the line being edited, so the
            // rendered-look styling must not land there.
            if (selection.ranges.some((r) => r.from <= line.to && r.to >= line.from)) continue;
            marks.push(lineMark.range(line.from));
            lastMarked = line.from;
        }
    }
    return Decoration.set(marks);
}
```

Recompute on `docChanged || viewportChanged || selectionSet` — don't narrow to `docChanged`, because moving the caret onto a line swaps the widget back for raw Markdown **without changing a character**.

**③ Live Preview, inside an Obsidian widget (`.image-embed`, rendered links) — DOM marking is safe here**, precisely because `WidgetView.ignoreMutation()` returns `true`: CodeMirror treats widget innards as foreign territory and never re-syncs their attributes. Worth doing when a line class would be too coarse (a line can hold a bookmark *and* an ordinary image). **But a `ViewPlugin`'s `update()` runs *before* CodeMirror syncs its DOM**, so marking there reads the previous render — queue it into the measure phase, whose `write` callback is the phase meant for DOM writes:

```typescript
export const bookmarkMarkers = ViewPlugin.fromClass(
    class {
        decorations: DecorationSet;
        constructor(private readonly view: EditorView) {
            this.decorations = bookmarkLines(view);
            this.scheduleEmbedPass();
        }
        update(update: ViewUpdate): void {
            if (update.docChanged || update.viewportChanged || update.selectionSet)
                this.decorations = bookmarkLines(update.view);
            this.scheduleEmbedPass();
        }
        private scheduleEmbedPass(): void {
            this.view.requestMeasure({
                key: this,                                   // dedupe: one pass per cycle
                read: () => null,
                write: () => syncEmbedMarks(this.view.contentDOM),
            });
        }
    },
    { decorations: (markers) => markers.decorations }
);

// Remove-then-add, not add-only: an element can outlive the content that earned its mark.
function syncEmbedMarks(root: Element): void {
    for (const marked of Array.from(root.querySelectorAll(`.${EMBED_CLASS}`)))
        if (!marked.querySelector(ICON_SELECTOR)) marked.removeClass(EMBED_CLASS);
    for (const icon of Array.from(root.querySelectorAll(ICON_SELECTOR)))
        icon.closest(".image-embed")?.addClass(EMBED_CLASS);
}
```

Cost check: each pass is two `querySelectorAll` calls over the viewport (CodeMirror keeps only visible lines in the DOM) — precisely the work `:has()` was making the engine redo on every invalidation. `classList.add` of a class that is already there mutates nothing, so the steady state produces no mutation records and no loop.

> Don't reach for `docViewUpdate()` (the `PluginValue` hook that *is* called after the DOM sync): it arrived in @codemirror/view 6.36, and the CodeMirror build a given Obsidian version bundles is not the one in your `devDependencies`. `requestMeasure` has been there since 6.0.
>
> Specificity check after the rewrite: `a.external-link:has(> img[src^="…"])` is (0,2,2), while `a.external-link.better-link-display-bookmark` is (0,2,1) — one notch lower. It still wins against Obsidian's own `a.external-link` rules on source order (plugin CSS loads after `app.css`), but if a rule stops taking effect, this is the first thing to check.
>
> How to check an ownership question like this for yourself, instead of guessing: the answer is in `node_modules/@codemirror/view/dist/index.js` — `grep -n "observeOptions" -A8` for what the observer watches, `grep -n "readMutation(rec)" -A8` for what it does with a record, and `grep -n "ignoreMutation" -A3` for which view classes opt out (`WidgetView` and `BlockWidgetView` return `true`; the base `ContentView`, and therefore `LineView`, returns `false`).

## Checking JS/TS for inline style assignment

```bash
grep -n '\.style\.' src/*.ts
```

Any hit that assigns a value (`.style.color =`, `.style.display =`, `.style.cssText =`) is a violation — the bot's message is `Sets styles directly instead of using CSS classes, setCssProps, or setCssStyles`. Use a CSS class for static values; for dynamic CSS properties use `setCssProps` (custom properties) or `setCssStyles` (standard properties):

```typescript
// ❌
el.style.cssText = 'position: absolute; top: 0';
el.style.color = 'red';
el.style.display = isReading ? '' : 'none';

// ✅ static — toggle a class instead of a direct property assignment
el.addClass('my-positioned-el');
el.toggleClass('is-hidden', !isReading);

// ✅ dynamic custom property
setCssProps(el, { '--offset-top': `${y}px` });

// ✅ dynamic standard property (when a class can't express the value)
setCssStyles(el, { display: isReading ? '' : 'none' });
```

## Checking JS/TS for innerHTML assignment

**Submission bot messages (both can fire on the same line):** `Unsafe assignment to innerHTML` (Error) and `Do not write to DOM directly using innerHTML/outerHTML property` (Warning).

`el.innerHTML = ...` is flagged twice, for two different reasons, but fixed by the same change: (1) the Obsidian rule bans `innerHTML`/`outerHTML` outright regardless of content, and (2) `HTMLElement.innerHTML`'s setter type is `string`, so assigning any non-literal (a template built from another `any`/untyped value, a variable typed loosely) also trips generic TS `no-unsafe-assignment`. A hardcoded SVG/HTML string literal still trips (1) even though it can't trip (2).

```typescript
// ❌ — src/layout/EditorStatusManager.ts:25 (real example from this repo)
const LOCK_SVG = `<svg xmlns="..." ...>...</svg>`;
this.statusBarItem.innerHTML = LOCK_SVG;

// ✅ — parse the SVG once and clone the node instead of re-assigning innerHTML
const LOCK_SVG_EL = new DOMParser().parseFromString(LOCK_SVG, 'image/svg+xml').documentElement;
this.statusBarItem.appendChild(LOCK_SVG_EL.cloneNode(true));
```

For plain text/element trees (the common case), skip the parser entirely and build with Obsidian's DOM helpers:

```typescript
// ❌
el.innerHTML = `<span class="label">${text}</span>`;

// ✅
el.createSpan({ cls: 'label', text });
```

```bash
grep -rn '\.innerHTML\s*=\|\.outerHTML\s*=' src/ main.ts
```

## Settings UI 规范

Settings 标签页的分组标题必须用 Obsidian Setting API，不能直接创建 HTML 标题元素。

```bash
grep -n "createElement\|\.createEl\b" src/settings.ts | grep -i "h[1-6]\|heading"
```

| 违规 | Fix |
|------|-----|
| `containerEl.createEl('h2', { text: 'Group' })` | `new Setting(containerEl).setName('Group').setHeading()` |
| `const h = document.createElement('h3')` | `new Setting(containerEl).setName('...').setHeading()` |

```typescript
// ❌
const heading = containerEl.createEl('h2', { text: 'Upload settings' });

// ✅
new Setting(containerEl).setName('Upload settings').setHeading();
```

## Obsidian DOM API Rules (submission bot 检查)

Obsidian 插件审核机器人会扫描以下模式：

### createElement → Obsidian 全局函数

| 违规 | Fix |
|------|-----|
| `document.createElement('div')` | `createDiv({ cls: 'my-class' })` |
| `document.createElement('span')` | `createSpan({ cls: 'my-class' })` |
| `document.createElement('img')` | `createEl('img')` |
| `document.createElement('button')` | `createEl('button')` |
| `document.createElement('label')` | `createEl('label', { cls: '...' })` |
| `document.createElement('input')` | `createEl('input', { cls: '...', type: 'checkbox' })` |
| `document.createElement('canvas')` | `createEl('canvas')` |

`createDiv` / `createSpan` / `createEl` 均为 Obsidian 全局函数，无需 import。`DomElementInfo` 支持 `{ cls, text, type, attr }` 等属性，可在创建时一步设置，省去单独赋值。

**目标父节点不是 workspace 里的某个 `containerEl`，而是 `document.head`/`document.body` 本身时，用父节点自己的 `.createEl()` 方法，而不是裸的全局 `createEl()`。** 全局 `createDiv()`/`createEl()` 创建的是一个*游离*节点，仍需手动 `appendChild` 才能挂上树；Obsidian 把同名方法也挂在了 `HTMLElement.prototype` 上（包括 `document.head`/`document.body` 这两个原生元素），调用 `parentEl.createEl(tag, info)` 会创建并**自动 append** 为 `parentEl` 的子节点，一步到位。往 `<head>` 里插入运行时样式表（主题 CSS、动态注入的 `<style>`）是这个模式最常见的场景：

```typescript
// ❌ — src/core/ThemeLoader.ts:43（实测，obsidian-minimalism-ui，2026-07-21）
const style = activeDocument.createElement('style');
style.setAttribute('data-my-theme', name);
style.textContent = css;
activeDocument.head.appendChild(style);

// ✅ — activeDocument.head 本身有 createEl，一次调用完成"创建+设属性+设内容+挂载"
activeDocument.head.createEl('style', {
    attr: { 'data-my-theme': name },
    text: css,
});
```

**⚠️ 2026-07-22 更正——上面 `<style>` 那个例子的"修复"是错的，只是换了创建方法，没有解决真正的违规。** 同一处代码（`ThemeLoader.ts:43`）按上面的建议从 `document.createElement('style')` 改成 `activeDocument.head.createEl('style', ...)` 后提交审核，被官方判定为一条全新的、**Error 级别**（直接拒绝发布）的违规：

```
Creating and attaching "style" elements is not allowed. For loading CSS, use a "styles.css" file
instead, which Obsidian loads for you.
```

**这条规则和"该用 `createEl` 还是 `document.createElement`"完全无关——Obsidian 插件审核禁止运行时创建并挂载*任何* `<style>` 元素，不管用哪个 DOM API 创建、不管挂在 `<head>` 还是别处。** `createEl('style', ...)` 只是解决了"该用哪个方法创建 DOM 节点"这个（次要的）风格问题，但"创建一个 `<style>` 节点并把它接到文档树上"这个动作本身就是被禁止的，无论方法多干净都一样违规。真正的修复方向是**把运行时才决定注入的 CSS 改为随 `styles.css` 静态分发**：

```typescript
// ❌ — 无论用 document.createElement 还是 activeDocument.head.createEl，
// "运行时创建并挂载 <style> 元素"这个动作本身就违规
export class ThemeLoader {
    apply() {
        this.remove();
        const css = THEME_CSS[this.settings().theme];
        activeDocument.head.createEl('style', { attr: { 'data-my-theme': name }, text: css });
    }
    remove() {
        activeDocument.head.querySelectorAll('style[data-my-theme]').forEach(el => el.remove());
    }
}

// ✅ — 主题 CSS 的源码（theme/<name>/<name>.css）已经收敛在 body.minimalism-ui-theme-<name>
// 选择器下，多个主题的规则可以同时静态存在于同一份 styles.css 里、互不覆盖——运行时
// 只需要切 body 上的类名来"选中"哪一段生效，完全不需要注入/移除任何 <style> 节点
export class ThemeLoader {
    apply() {
        this.remove();
        const name = this.settings().theme;
        if (name) activeDocument.body.classList.add(`minimalism-ui-theme-${name}`);
    }
    remove() {
        const cls = activeDocument.body.classList;
        Array.from(cls).filter(c => c.startsWith('minimalism-ui-theme-')).forEach(c => cls.remove(c));
    }
}
```

**把 CSS 折进静态 `styles.css`的构建步骤**：如果原来的动机是"多主题/多套 CSS 太大，不想全部塞进一份 CSS"，唯一合规的出路是让构建脚本把各主题 CSS 拼接进仓库里那份会被提交、会被 CI 直接打进 release 的 `styles.css`（用一对标记注释包住生成区间，脚本每次整段替换，保持幂等；手改仍然改 `theme/<name>/<name>.css` 源文件，不要手改标记之间的内容）：

```javascript
// scripts/generate-theme-assets.mjs（构建前置步骤，dev/build 都会跑）
const BEGIN = '/* GENERATED-THEME-CSS:BEGIN */', END = '/* GENERATED-THEME-CSS:END */';
const stylesSrc = readFileSync('styles.css', 'utf8');
const i = stylesSrc.indexOf(BEGIN), j = stylesSrc.indexOf(END);
const block = themeNames.map(n => readFileSync(`theme/${n}/${n}.css`, 'utf8').trim()).join('\n\n');
writeFileSync('styles.css', stylesSrc.slice(0, i + BEGIN.length) + '\n\n' + block + '\n\n' + stylesSrc.slice(j));
```

**字体不受这条规则影响，不用跟着改**：`FontFace` API（`new FontFace(...)` + `document.fonts.add(...)`）注册的是字体，不创建/挂载 `<style>`/`<link>` 元素，这条禁令管不到它——`FontLoader` 之类通过 base64 内嵌 + `FontFace` 加载字体的机制原样保留即可，只有"把 CSS 规则文本塞进 `<style>` 标签再插入文档"这一类操作会被这条规则命中。

**排查方法**：这条规则和"是否用了 `document.createElement`"这个信号完全脱钩——`createEl('style', ...)` 一样会中招。改查*创建目标是不是 `'style'` 这个 tag*，不管调用的是哪个 DOM 方法：

```bash
grep -rn "createElement(.style.\|createEl(.style.\|createEl(\"style\"" src/ main.ts
```

命中任何一行，无论用的是 `document.createElement` 还是 Obsidian 的 `createEl`，都要按上面的思路整体挪到静态 `styles.css`，不能只换 API。

```bash
grep -n "document\.createElement" src/*.ts
```

```typescript
// ❌
const el = document.createElement('div');
el.className = 'foo';
const span = document.createElement('span');
span.textContent = 'hello';
const input = document.createElement('input');
input.type = 'checkbox';

// ✅
const el = createDiv({ cls: 'foo' });
const span = createSpan({ text: 'hello' });
const input = createEl('input', { type: 'checkbox' });
```

### 裸用 `document` / 定时器 / `instanceof` → popout 兼容写法

Obsidian 支持笔记在独立弹出窗口（popout window）中打开。跨窗口时，裸用全局 `document`、定时器或 `instanceof` 会绑定到错误的窗口对象，导致行为异常或类型判断失效。审核机器人按三类给出**不同**的修复方向——关键陷阱：**DOM 访问用 `activeDocument`，定时器却要用 `window`，两者不可混用**。

**1. DOM / 事件访问 → `activeDocument.*`**（跟随活动窗口）

| 违规 | Fix |
|------|-----|
| `document.body.appendChild(el)` | `activeDocument.body.appendChild(el)` |
| `document.addEventListener(...)` | `activeDocument.addEventListener(...)` |
| `document.removeEventListener(...)` | `activeDocument.removeEventListener(...)` |
| `document.querySelector(...)` | `activeDocument.querySelector(...)` |

**2. 定时器函数 → `window.*`**（不是 `activeWindow`，也不是裸用）

> 机器人原文：`Use 'window.setTimeout()' instead of 'activeWindow.setTimeout()'. Timer functions should use 'window'.` · `Use 'window.requestAnimationFrame()' instead of 'requestAnimationFrame()' for popout window compatibility.`

| 违规（裸用 **或** `activeWindow.`） | Fix |
|------|-----|
| `setTimeout(fn, ms)` / `activeWindow.setTimeout(...)` | `window.setTimeout(fn, ms)` |
| `clearTimeout(id)` / `activeWindow.clearTimeout(...)` | `window.clearTimeout(id)` |
| `setInterval(...)` / `clearInterval(...)` | `window.setInterval(...)` / `window.clearInterval(...)` |
| `requestAnimationFrame(fn)` / `activeWindow.requestAnimationFrame(...)` | `window.requestAnimationFrame(fn)` |
| `cancelAnimationFrame(id)` | `window.cancelAnimationFrame(id)` |

⚠️ **定时器是 DOM 规则的例外**：DOM 用 `activeDocument`，但定时器一律 `window.`。两个原因：① `window.setTimeout` 命中 DOM lib 重载返回 `number`（装了 `@types/node` 时裸用 `setTimeout` 返回 `NodeJS.Timeout`，类型不符）；② timer 不应绑定到随时可能被关闭的 popout 窗口。

**3. DOM 类型判断 → `node.instanceOf(Type)`**（跨窗口安全）

> 机器人原文：`Use '.instanceOf(Element)' instead of 'instanceof Element' for cross-window safe type checking.`

`instanceof Element` 在 popout 中失效——每个窗口有各自的 `Element` 构造函数。Obsidian 在 `Node` 上挂了 `instanceOf<T>(type): this is T` 方法（无需 import）：

| 违规 | Fix |
|------|-----|
| `node instanceof Element` | `node.instanceOf(Element)` |
| `node instanceof HTMLElement` | `node.instanceOf(HTMLElement)` |

只对 **DOM 节点**用 `.instanceOf()`。`instanceof TFile` / `instanceof WorkspaceLeaf` 等 Obsidian 模型类不是 DOM 节点，**保持 `instanceof` 不变**。

`activeDocument` / `activeWindow` 均为 Obsidian 全局，无需 import。

```bash
# 1. 裸用 document（非 activeDocument）
grep -rn "\bdocument\." src/ main.ts | grep -v "activeDocument\."
# 2. 定时器：裸用或 activeWindow（都要改成 window.）
grep -rEn "activeWindow\.(setTimeout|clearTimeout|setInterval|clearInterval|requestAnimationFrame|cancelAnimationFrame)|(^|[^.])(setTimeout|clearTimeout|setInterval|clearInterval|requestAnimationFrame|cancelAnimationFrame)\(" src/ main.ts | grep -v "window\."
# 3. DOM instanceof（Element / HTMLElement / Node）
grep -rn "instanceof \(Element\|HTMLElement\|Node\)" src/ main.ts
```

```typescript
// ❌
document.body.appendChild(overlay);
activeWindow.setTimeout(() => overlay.remove(), 300);
requestAnimationFrame(() => layout());
if (node instanceof Element) { /* ... */ }

// ✅
activeDocument.body.appendChild(overlay);
window.setTimeout(() => overlay.remove(), 300);
window.requestAnimationFrame(() => layout());
if (node.instanceOf(Element)) { /* ... */ }
```

## Checking for unnecessary console logging

**Submission bot warning:** `Avoid unnecessary logging to console` — https://docs.obsidian.md/Plugins/Releasing/Plugin+guidelines#Avoid+unnecessary+logging+to+console

```bash
grep -n "console\.\(log\|info\|debug\)" src/*.ts main.ts
```

The guideline targets *routine* logging — a line that fires on every normal call, not just on failure. `console.error`/`console.warn` for an actual failure (one the developer console needs so a bug report is diagnosable) are not what this flags; only remove those if the call site has no error to report.

```typescript
// ❌ — fires on every request, tells the developer nothing they couldn't already infer
export function logInfo(message: string, detail?: unknown): void {
	console.info(`${PREFIX} ${message}`, detail);
}
logInfo(`requesting ${endpoint}`);

// ✅ — delete the info-level logger and its call sites entirely; keep warn/error
// for the failure branches that still need a diagnosable console trail
```

If a logging helper (like a `logInfo` wrapper) ends up with no remaining callers after deleting the routine calls, delete the helper too rather than leaving dead code behind.

## Checking for hardcoded `.obsidian` config directory

**Submission bot warning:** `Obsidian's configuration folder is not necessarily .obsidian, it can be configured by the user. Use Vault#configDir to get the current value`

```bash
grep -n '"\.obsidian' src/*.ts main.ts
grep -n "'\.obsidian" src/*.ts main.ts
```

Users can rename the hidden config folder in Obsidian's settings, so any path built with a literal `.obsidian` segment silently breaks for them. Build it from `Vault#configDir` instead:

```typescript
// ❌ — assumes the config folder is always named ".obsidian"
const DEFAULT_CACHE_DIR = ".obsidian/plugins/my-plugin/cache";

// ✅ — a function of the actual configDir, read from the running vault
function defaultCacheDir(configDir: string): string {
	return `${configDir}/plugins/my-plugin/cache`;
}
// called as defaultCacheDir(this.app.vault.configDir)
```

Where the hardcoded path was a module-level constant computed before any `App` instance existed (e.g. a settings-defaults factory), turn it into a function that takes `configDir` as a parameter and thread `this.app.vault.configDir` through from every call site — including the plugin's own `onload`/`loadSettings` and any settings-tab fallback that rebuilds the default when a user-editable field is cleared.

## Checking for `Vault.trash()` instead of `FileManager.trashFile()`

**Submission bot warning:** `Use 'FileManager.trashFile()' instead of 'Vault.trash()' to respect the user's file deletion preference.`

```bash
grep -n "vault\.trash(" src/*.ts main.ts
```

`Vault.trash(file, system)` only gives a binary choice — pass `system: true` for the OS trash, `false` for `.trash` inside the vault — and ignores whatever the user actually configured in Settings → Files & Links → "Deleted files". `FileManager.trashFile(file)` reads that setting itself (system trash / vault trash / permanent delete) and does the right thing, so any plugin-initiated delete should go through it:

```typescript
// ❌ — hardcodes "system trash", ignoring the user's configured deletion behavior
await app.vault.trash(file, true);

// ✅ — defers to Settings → Files & Links → "Deleted files"
await app.fileManager.trashFile(file);
```

`FileManager.trashFile` requires Obsidian **1.6.6** (`no-unsupported-api` fires below that) — if `manifest.json`'s `minAppVersion` is older, this is the same "fix the API vs. raise the floor" tradeoff as the `getSettingDefinitions()` migration below: either bump `minAppVersion` to `1.6.6`+ (and `versions.json`'s entry for the next release) and switch the call, or confirm with the maintainer that keeping `Vault.trash()` at the current `minAppVersion` is acceptable and leave the Warning unaddressed (it's non-blocking). Don't silently keep `Vault.trash()` while claiming the bot finding is fixed — a stale comment explaining "why not trashFile" (like the one this rule was discovered from, which cited a `minAppVersion` that no longer matched reality once the floor had already moved for other reasons) is worse than no comment, because the next person trusts it instead of re-checking.

## TypeScript 类型安全

### `unknown` 与其他类型组成联合类型 → `no-redundant-type-constituents`

机器人原文：`'unknown' overrides all other types in this union type.` `unknown` 是所有类型的顶类型，`X | unknown` 在结构上恒等于 `unknown`——联合里的其他成员（哪怕是 `null`、某个具名接口）都被 `unknown` 吞掉，写出来没有任何额外的类型信息，纯属误导读者。常见于给一个"可能拿不到具体类型、干脆返回 unknown"的私有辅助方法顺手加了 `| null`：

```typescript
// ❌ — src/right-sidebar/RightSidebarButtonManager.ts:597（实测，obsidian-minimalism-ui，2026-07-12）
// unknown | null 在类型层面等价于 unknown，| null 是死代码，读者会误以为返回值区分"有值/无值"
private findExistingRightSidebarParent(): unknown | null {
    const { rightSplit } = this.app.workspace;
    if (!rightSplit) return null;
    // ...
    return found ? (found as unknown as { parent: unknown }).parent : null;
}

// ✅ — 直接删掉 `| null`，返回类型如实反映"unknown 本身已经涵盖 null"
private findExistingRightSidebarParent(): unknown {
    // ...
}
```

> 判断信号：报告的联合类型字面量里出现 `unknown`（不是 `any`——`any` 不会吞并 `unknown`/`never` 之外的成员，只有 `unknown` 才有这种"覆盖一切"的效果），修法永远是删掉其余成员，只保留 `unknown`。函数体内 `return null`/`return someValue` 不需要跟着改——`null`、具体对象都能隐式赋值给 `unknown`，删掉联合里多余的成员只影响*声明的返回类型*，不影响函数体的合法性。

### 避免 `as TFile` 强制转换 → 用 `instanceof TFile` 收窄

机器人原文：`Avoid casting to 'TFile'. Use an 'instanceof TFile' check to safely narrow the type.` `getAbstractFileByPath` 返回 `TAbstractFile | null`，强转 `as TFile` 会绕过类型系统（实际可能是 `TFolder`），运行时不安全。正确做法是先 `instanceof TFile` 收窄，使变量在使用处**已经是** `TFile`，无需断言。

> 注意：这与 popout DOM 规则里的 `node.instanceOf(Element)` 不同——`TFile`/`TFolder`/`WorkspaceLeaf` 是 Obsidian 模型类、不是 DOM 节点，用普通 `instanceof`。

```typescript
// ❌ — file 是 TAbstractFile | null，强转掩盖了它可能不是 TFile
let file = this.app.vault.getAbstractFileByPath(path);
if (!(file instanceof TFile)) {
    file = await this.app.vault.create(path, "");   // create 返回 TFile
}
await this.app.workspace.getLeaf(false).openFile(file as TFile);

// ✅ — 用一个声明为 TFile 的变量承接 create 的结果，全程无断言
let file = this.app.vault.getAbstractFileByPath(path);
let target: TFile;
if (file instanceof TFile) {
    target = file;
} else {
    target = await this.app.vault.create(path, "");
}
await this.app.workspace.getLeaf(false).openFile(target);
```

```bash
grep -rn "as TFile\b\|as TFolder\b" src/ main.ts
```

### `loadData()` 返回 `any` 导致不安全赋值

`Plugin.loadData()` 返回 `Promise<any>`，直接赋值给强类型字段会触发 `@typescript-eslint/no-unsafe-assignment`：

```typescript
// ❌ — Unsafe assignment of an `any` value
this.settings = Object.assign({}, DEFAULT_SETTINGS, await this.loadData());

// ✅ 加类型断言
this.settings = Object.assign({}, DEFAULT_SETTINGS, (await this.loadData()) as Partial<MyPluginSettings>);
```

`parseYaml()` 同理返回 `any`，直接赋值/return 触发同一条。先收到 `unknown`，再 `typeof === "object"` 收窄后断言：

```typescript
// ❌ — Unsafe assignment / return of an `any` value
const parsed = parseYaml(trimmed);
if (parsed && typeof parsed === "object") return parsed as BlockConfig;

// ✅ 显式收为 unknown，收窄后再断言
const parsed: unknown = parseYaml(trimmed);
if (parsed && typeof parsed === "object") return parsed as BlockConfig;
```

`JSON.parse()`（如读取 vault adapter 上缓存的 JSON 文件）是完全同样的模式——返回值是 `any`，直接赋值给 `const` 就触发 `no-unsafe-assignment`，即使后面只是喂给一个 `value is T` 类型守卫：

```typescript
// ❌ — Unsafe assignment of an `any` value
const parsed = JSON.parse(await adapter.read(path));
this.cache = isValidShape(parsed) ? parsed : {};

// ✅ 标注为 unknown——类型守卫本来就接受 unknown，不需要额外断言
const parsed: unknown = JSON.parse(await adapter.read(path));
this.cache = isValidShape(parsed) ? parsed : {};
```

### `.bind()` / `.call()` / `.apply()` 返回 `any` → 开 `strictBindCallApply`

未开 `strictBindCallApply` 时，`Function.prototype.bind` 的类型签名返回 `any`，于是 `const f = obj.method.bind(obj)` 整条链都变 `any`，引发一整簇 `no-unsafe-assignment` / `no-unsafe-call` / `no-unsafe-argument`（最常见于 monkey-patch：保存原方法 → 调用 → 存进 Map）。逐处加断言治标不治本——**根因在 tsconfig**。

**根因修复**：在 `tsconfig.json` 开启该选项（可独立开启，不必上全量 `strict`），所有 `.bind/.call/.apply` 立即获得正确类型，一次消除整簇告警，业务代码一行不用改：

```jsonc
// tsconfig.json → compilerOptions
"strictBindCallApply": true
```

```typescript
// ❌ 无 strictBindCallApply：bind 返回 any
const origBack = history.back.bind(history);   // any
this.patches.set(leaf, origBack);              // no-unsafe-argument
origBack();                                     // no-unsafe-call

// ✅ 开启后：origBack 自动推断为 () => void，无需改任何业务代码
```

> 判断信号：若审核报告里 `no-unsafe-*` 几乎都落在 `xxx.bind(...)` 的赋值/调用处，几乎可断定就是这个开关没开。开启后先 `grep -rn "\.bind(\|\.call(\|\.apply(" src/ main.ts` 核对各处 `this` 绑定正确，再 `tsc -noEmit` 确认无新错误——本选项只会让类型更精确，极少引入新报错。

### `async` 函数传给 `addEventListener` → `no-misused-promises`

把 `async` 箭头函数直接交给期望 `void` 返回值的 API（`addEventListener`、`setTimeout` 等），会触发 `Promise returned in function argument where a void return was expected`。注意 `new Setting().onChange(async ...)` 等 Obsidian API **接受** Promise 回调，不会被标记；只有 DOM 的 `addEventListener` 这类才会。

```typescript
// ❌
el.addEventListener('change', async () => {
    await this.plugin.saveSettings();
});

// ✅ 去掉 async，用 void 触发 fire-and-forget
el.addEventListener('change', () => {
    void this.plugin.saveSettings();
});
```

> 去掉 `async` 后函数体内不能再用 `await`——把那一行 `await fn()` 改成 `void fn()`（同一次编辑里改完，否则是语法错误）。

### 多余的类型断言 → `no-unnecessary-type-assertion`

`x as T` 中，若 `x` 经控制流收窄后**已经**是 `T`，断言就是多余的，触发 `This assertion is unnecessary since it does not change the type of the expression`。直接删掉断言：

```typescript
// ❌ — lang 在 else 分支已被收窄为 'zh' | 'en'，正好等于 Lang
type Lang = 'zh' | 'en';
function setLang(lang: 'auto' | 'zh' | 'en') {
    langOverride = lang === 'auto' ? null : lang as Lang;
}

// ✅
langOverride = lang === 'auto' ? null : lang;
```

**同类陷阱：库函数返回类型已经是目标类型**——不只是控制流收窄能让断言变多余，第三方库的类型声明本身已精确到目标类型时，包一层 `as` 同样多余：

```typescript
// ❌ — fflate 的 strToU8()/zipSync() 类型声明已返回 Uint8Array<ArrayBuffer>，
// 接收方（Zippable 的 value、Promise<{ zip: Uint8Array }> 的字段）本就接受这个类型
files[name] = strToU8(html) as Uint8Array;
const zip = zipSync(files, { level: 6 }) as Uint8Array;

// ✅
files[name] = strToU8(html);
const zip = zipSync(files, { level: 6 });
```

不确定库的真实返回类型时，别猜——直接查 `node_modules/<pkg>/**/*.d.ts` 里的函数签名，或用 TS 官方 API 现场核实（见下方"核实陈旧报告"）。

**同类陷阱的 Obsidian 官方 API 版本（实测案例，obsidian-minimalism-ui，2026-11-06）**：`leaf.getViewState().state as Record<string, unknown> | undefined` 这类写法，先查 `node_modules/obsidian/obsidian.d.ts` 里 `ViewState` 接口的 `state` 字段声明——它本身就是 `state?: Record<string, unknown>`，断言目标与官方声明完全一致，断言不改变任何类型，直接删掉：

```typescript
// ❌ — ViewState.state 在 obsidian.d.ts 里已声明为 Record<string, unknown> | undefined
const state = leaf.getViewState().state as Record<string, unknown> | undefined;

// ✅
const state = leaf.getViewState().state;
```

遇到类似断言，先 `grep -n "interface ViewState" -A5 node_modules/obsidian/obsidian.d.ts`（或任意涉及的 Obsidian 接口）核实官方字段类型，而不是假设它是 `any` 后去猜断言是否必要。

**⚠️ 第三类陷阱——`as` 断言本身就是让类型"看起来不需要断言"的原因，删掉后类型会真的变宽（实测案例，obsidian-image-cluster，2026-07-10）**：`Element.prototype.closest<E extends Element = Element>(selectors: string): E | null` 是泛型方法，`E` 默认收窄为 `Element`。当写成 `el.closest(".cm-editor") as HTMLElement | null` 时，TypeScript 会把 `as` 右边的类型当作**上下文类型**反向喂给 `closest<E>` 的泛型推断（这是 TS 对括在 `as` 里的表达式做上下文类型标注的通用行为，不是 `closest` 特有的），于是 `E` 被推成 `HTMLElement`——断言写出来之后，表达式自己的类型就已经等于断言目标，`no-unnecessary-type-assertion` 因此判定"断言不改变类型"而报多余。**但这是循环论证式的假象**：断言之所以"不改变类型"，恰恰是因为断言本身才是让 `E` 被推成 `HTMLElement` 的原因——按标准做法直接删掉 `as`，`E` 会失去这个上下文提示，退回默认值 `Element`，类型真的变宽了：

```typescript
// ❌ — 机器人报 no-unnecessary-type-assertion，但直接删掉 as 是错的
const editorRoot = el.closest(".cm-editor") as HTMLElement | null;
// 若真的删掉 as：editorRoot 的类型退化为 Element | null（closest<E> 失去上下文推断依据）
// 而下面这行要求形参是 HTMLElement，Element 不能赋值给它，产生真实的 TS2345 错误
EditorView.findFromDOM(editorRoot);   // declare findFromDOM(dom: HTMLElement): EditorView | null

// ✅ — 用显式类型实参替代 as，直接让 closest 在类型层面就返回 HTMLElement | null，
// 断言的"上下文类型反哺泛型推断"这条隐藏路径完全不再涉及，机器人和本地 tsc 都满意
const editorRoot = el.closest<HTMLElement>(".cm-editor");
```

**判断信号**：报告位置是一个「泛型方法调用外面包一层 `as`」（不是简单变量/字面量），且该泛型方法有 `<T extends Base = Base>` 形式的默认类型参数——直接删断言前，先手写一个不带 `as`、不带任何外部上下文类型的最小复现文件，用上面「核实报告是否陈旧」一节的 TS Compiler API 脚本实际打印删除断言后的类型，确认是否真的收窄到位；若变宽了，改用 `方法名<T>(...)` 显式类型实参，而不是删掉断言。`Element.closest`/`querySelector`/`querySelectorAll` 等 DOM 方法均属此类（`querySelector<E extends Element = Element>(selectors: string): E | null`），符合就用同一个修法。

### 核实 no-unsafe-* / no-unnecessary-type-assertion 报告是否已过时

审核报告里的行号对应的是**提交时的某个历史 commit**，如果这之后已经给相关接口补上了显式类型（例如给 `parseYaml()` 的结果定义了具体 interface 并 `as` 过一次），报告里那几行在当前代码里可能已经不再是 `any` 了。逐条盲目"修复"已经安全的代码没有意义，先核实：

```bash
# 用项目自己的 tsconfig + 完全一致的 TypeScript 版本，查询某个表达式的实际推断类型
cat > /tmp/check-types.mjs <<'EOF'
import ts from "typescript";
const parsed = ts.parseJsonConfigFileContent(
  ts.readConfigFile("tsconfig.json", ts.sys.readFile).config, ts.sys, "."
);
const program = ts.createProgram(["src/render/base-renderer.ts"], parsed.options);
const checker = program.getTypeChecker();
const sf = program.getSourceFile("src/render/base-renderer.ts");
function visit(node) {
  if (ts.isVariableDeclaration(node) && node.name.getText() === "view") {
    console.log(checker.typeToString(checker.getTypeAtLocation(node.name)));
  }
  ts.forEachChild(node, visit);
}
visit(sf);
EOF
cp /tmp/check-types.mjs . && node check-types.mjs; rm check-types.mjs
```

若脚本打印出的类型是具体的 interface/union 而不是 `any`，说明该处已经被之前的类型标注修复过，报告是陈旧的——无需改代码，但值得在下一次提交审核前确认问题确实不再出现。

**轻量核实法（不用起 TS API 脚本）**：当报告落在一次简单的方法/属性调用上（`x.includes(...)`、`x.push(...)`、`x.filter(...)` 这类），不必上 TS Compiler API——直接 grep 出该变量/字段的声明处，看它是不是已经有具体类型（`string[]`、`WorkspaceLeaf[]` 等具名类型），而不是 `any` 或未标注：

```bash
# 例：报告称 src/single-page/LeafCache.ts:52 的 no-unsafe-call
grep -n "private queue" src/single-page/LeafCache.ts   # -> private queue: WorkspaceLeaf[] = [];
```

若声明已经是具名类型，`tsc -noEmit` 也不报错，那么该行当前是类型安全的，报告陈旧，跳过。**已核实为陈旧的一批**（2026-07-07 报告，对照当前代码全部已有具名类型，`pnpm build` 的 `tsc -noEmit` 通过）：

| 报告位置 | 变量/字段声明 | 结论 |
|---------|---------------|------|
| `src/SettingTab.ts:100` | `listThemes(): string[]`（`main.ts`），`names.includes(...)` | `names` 是 `string[]`，非 `any` |
| `src/onboarding/OnboardingManager.ts:233` | `onboardingDone: string[]`（`src/core/settings.ts`），`done.includes(label)` | `done` 是 `string[]` |
| `src/onboarding/OnboardingManager.ts:272` | 同上，`settings.onboardingDone.includes(active.label)` | 同上 |
| `src/single-page/LeafCache.ts:52` | `private queue: WorkspaceLeaf[]`，`rootLeaves.includes(l)` | `rootLeaves: WorkspaceLeaf[]`，非 `any` |

> 这批例子的共性：都是 `Array.prototype` 方法（`includes`/`push`/`filter`）调用在一个早已声明为具名数组类型的字段/变量上——不存在这几个方法签名退化为 `any` 的原因，说明审核报告采样的是更早（未加类型标注前）的历史 commit。核实后无需改代码，但下次提交前仍建议留意同一位置是否重新出现（例如该字段的声明被误改回宽类型）。

> **⚠️ 2026-07-10 更正——上面这整批"已核实为陈旧"的结论是错的。** Obsidian 官方审核在 2026-07-10 对同一仓库（`obsidian-minimalism-ui`）重新给出了几乎一模一样的报告：`SettingTab.ts:100`、`OnboardingManager.ts:233`、`OnboardingManager.ts:272`、`LeafCache.ts:52` 全部原样复现，外加 `RightSidebarButtonManager.ts` 六处新增命中——逐一核对后，六处新增命中同样全部落在 `.includes(` 调用上（`path.includes(...)` × 4、`this.leafOrder.includes(...)` × 1、以及一处 `const inStack = ... path.includes(...)` 的赋值）。**这批诊断三天后原样复现，且新文件里的命中同样 100% 落在 `.includes()` 上，说明"陈旧报告"的解释是错的——根因当时就存在，只是"轻量核实法"（只查字段声明是否具名数组类型）问的问题不对。** 真正的判断标准不是"这个数组变量有没有具名类型"，而是"`.includes()` 这个*方法本身*在这个项目的有效 `lib` 集合里是否真的被声明"——这是两个独立的问题，具名数组类型（`string[]`、`WorkspaceLeaf[]`）完全不能证明 `.includes` 方法存在，因为该方法来自 `lib.es2016.array.include.d.ts`，是否加载取决于 `tsconfig.json` 的 `"lib"` 字段，与数组元素类型无关。详见下方新增小节，附完整根因定位与验证过程。

### `.includes()` 等 ES2016+ 数组方法在审核环境里被判定为 `error` type——`tsconfig.json` 的 `"lib"` 差一版，靠 `@types/node` 意外补齐

**背景**：本仓库 `tsconfig.json` 长期写着 `"lib": ["ES6", "DOM"]`（`ES6` = `ES2015`）。`Array.prototype.includes()` 是 **ES2016** 才引入的方法（`lib.es2016.array.include.d.ts`），理论上在 `lib: ["ES6", "DOM"]` 下该方法根本不存在，任何 `.includes(...)` 调用都应该报 `TS2550: Property 'includes' does not exist on type 'T[]'`。但本仓库 `pnpm build`（`tsc -noEmit -skipLibCheck`）长期干干净净、从未报过这个错——这就是为什么连续两次审核报告（2026-07-07、2026-07-10）都被误判成"报告陈旧/已经类型安全"：本地验证工具（`tsc -noEmit`）看到的和审核机器人看到的是**两个不同的类型程序**。

**根因**：`package.json` 的 `devDependencies` 里有 `@types/node`，其 `node_modules/@types/node/index.d.ts` 顶部有一行：

```typescript
/// <reference lib="es2020" />
```

TypeScript 的行为是：只要程序里**任意一个**被包含的文件（不限于你自己的源码，@types 包的环境声明文件同样算数）带有 `/// <reference lib="..." />`，这个 lib 就会被并入整个程序的全局库集合——**无视 `tsconfig.json` 里 `"lib"` 数组的显式限制**。`@types/node` 默认对所有 `.ts` 文件全局生效（没有 `"types"` 字段收窄的情况下），于是它这一行 reference 把 `lib.es2020.*`（包含 `lib.es2016.array.include.d.ts`）悄悄塞进了程序，本地 `tsc` 看到的 `Array<T>.includes` 因此是合法方法，永远不报错。

**但审核机器人构建的类型检查程序显然没有走到这条隐藏路径**（大概率是 `@typescript-eslint` 对单文件/子集做类型解析时用了不同的 program，或流水线没有安装 `@types/node`），于是在审核环境里 `Array<T>` 只剩 `lib.es2015.core.d.ts` 定义的方法（外加 `obsidian.d.ts` 自己挂的 `contains`/`remove`/`first`/`last`/`shuffle`/`unique`/`findLastIndex`——**注意 `obsidian.d.ts` 全局扩展了 `Array<T>` 但没有加 `includes`**），`.includes` 是不存在的成员，访问它得到 `error` type，于是每一处 `.includes(...)` 调用同时触发 `no-unsafe-call`（调用一个 `error` 类型的"方法"）和（当结果被赋值/传参时）`no-unsafe-assignment`。

**独立复现验证**（不猜测，实测）：

```bash
# 1. 确认 lib.es2016.array.include.d.ts 只有在 @types/node 存在时才会被隐式拉入
npx tsc -noEmit -skipLibCheck --listFiles | grep es2016.array.include
# → 有输出：说明当前确实是靠 @types/node 的隐藏 reference 兜底

# 2. 物理移除 @types/node，模拟审核环境，用旧 lib 配置重测
mv node_modules/@types/node node_modules/@types/node.disabled
# （用一份临时 tsconfig，只把 "lib" 改回 ["ES6","DOM"]，其余字段不变）
npx tsc -p /tmp/tsconfig.old.json -noEmit -skipLibCheck 2>&1 | grep -c "does not exist on type"
# → 1（复现 TS2550，证实旧配置在没有 @types/node 时真的会炸）

# 3. 同样移除 @types/node，但 "lib" 显式写成 ["ES2016","DOM"]
npx tsc -p /tmp/tsconfig.fixed.json -noEmit -skipLibCheck 2>&1 | grep -c "does not exist on type"
# → 0（修复后不再依赖 @types/node 的隐藏兜底）

mv node_modules/@types/node.disabled node_modules/@types/node
```

**修复**（一行改动，正是本仓库 2026-07-10 采用的方案）：

```jsonc
// tsconfig.json → compilerOptions
// ❌ — ES6 = ES2015，Array.prototype.includes 是 ES2016 才有的方法；
// 本地 tsc 之所以不报错，纯粹是 @types/node 的 /// <reference lib="es2020" /> 意外兜底
"lib": ["ES6", "DOM"]

// ✅ — 显式声明到实际用到的最低版本；lib.es2016.d.ts 会级联引入 lib.es2015.d.ts，
// 不丢失原有的 ES2015 特性，且不再依赖任何第三方包的隐藏 reference
"lib": ["ES2016", "DOM"]
```

选 `"ES2016"` 而非更高版本的依据：先 grep 一遍项目实际用到的新特性，只升到覆盖这些用法的最低版本，不过度升级：

```bash
grep -rEn '\.includes\(|\.flat\(|\.flatMap\(|Object\.fromEntries|Promise\.allSettled|\.replaceAll\(|\.matchAll\(' src/ main.ts
```

若结果只有 `.includes(`，`"ES2016"` 就够；若还用了 `.flat()`/`.flatMap()`（ES2019）或 `Promise.allSettled`（ES2020）等，相应把 `"lib"` 提到对应版本（TS 的 `lib.esXXXX.d.ts` 都是级联引用前序版本，不需要手写多个条目）。

**如何判断"报告是否陈旧"这件事本身要不要重新调查**：如果同一批 `no-unsafe-call`/`no-unsafe-assignment` 诊断在两次相隔数天的审核里**逐字重复**（尤其是新增文件里的新命中，也精确落在同一个方法/模式上），这本身就是判断信号——真正陈旧的报告不会在无关代码新增时，用同一个方法名在新文件里稳定复现。此时应该反过来怀疑"之前判定为陈旧"的验证方法是否问错了问题（例如只检查了变量声明是否具名类型，而没有检查*调用的那个方法*是否真的在项目当前 `lib` 配置下存在），而不是继续假设审核机器人在用旧 commit。

**排查方法**：报告指向 `.includes()`/`.flat()`/`.at()`/`Object.fromEntries()` 等任何 ES2016 及更新版本引入的方法时，先查 `tsconfig.json` 的 `"lib"` 是否覆盖到对应版本；再用 `--listFiles | grep <lib文件名>` 确认这个 lib 文件在当前配置下**是否真的因为 `"lib"` 字段本身**被加载（而非靠某个 `@types/*` 包的隐藏 reference）。判断某个 `@types/*` 包是否存在这类隐藏 reference：

```bash
grep -rn 'reference lib=' node_modules/@types/*/index.d.ts node_modules/@types/*/*.d.ts 2>/dev/null
```

**⚠️ 陷阱——`tsc -noEmit` 通过不能排除 `no-unsafe-*`，尤其当报错文案是 "`error` typed value" 而非 "`any` typed value"**：这两种措辞对应完全不同的根因，务必先看清报告原文用的是哪个词：

- **`any` typed value** —— 真正的隐式/显式 `any`（`loadData()`、`.bind()`、`catch` 变量、无类型 SDK 等），本文档其余小节覆盖的都是这类。
- **`error` typed value** —— TS 类型检查器**根本没能解析出类型**，退化为内部 `error` type。当 `tsconfig.json` 开着 `skipLibCheck: true`（Obsidian 插件模板的默认值）时，这类错误在第三方 `.d.ts` 内部被**静默吞掉**——`tsc -noEmit` 干干净净、零报错，但 `@typescript-eslint`（走真正的 TS 类型检查器 API，不受 `skipLibCheck` 影响，或者说它检查的是使用处而非声明文件本身）能看穿这个 `error` type，逐一标记出所有用到该值的地方。**只跑 `tsc -noEmit` 通过，不足以证明报告陈旧**——必须实际起一次 typescript-eslint 才能确认。

  实测案例（**obsidian-link-trace**，2026-07-07）：`src/render/base-renderer.ts:570-593` 与 `src/publish/exporter.ts:384-394` 报了一整串 `no-unsafe-member-access`/`no-unsafe-assignment`/`no-unsafe-call`/`no-unsafe-argument`，全部原文都是 "**error** typed value"。当时 `package.json` 里 `typescript` 精确锁定在 `4.7.4`（Obsidian 官方脚手架的老默认值），而依赖 `fflate` 的 `.d.ts` 用了 `Uint8Array<ArrayBuffer>` 这种**泛型化的类型化数组**语法（TS 5.7 才支持——TS 4.7 里 `Uint8Array` 根本不是泛型）。`skipLibCheck` 让 `tsc` 对 `fflate/lib/index.d.ts` 自身不做校验，于是这个解析失败被吞掉、`pnpm typecheck` 通过；但每一处 `strToU8()`/`zipSync()` 的返回值在使用处都被判定为 `error` type，凡是赋值/成员访问/当参数传递的地方全部触发 `no-unsafe-*`。

  **真正的修复**：把 `typescript` 从 `4.7.4` 升级到一个既 ≥5.7（`fflate` 泛型语法能正常解析）又不触发更新版本引入的新 breaking change 的版本——本例中 TS 5.9 引入了 `ArrayBufferView<TArrayBuffer>` 更严格的泛型约束，导致 `Uint8Array<ArrayBufferLike>` 不能赋给 `BlobPart`/`ArrayBuffer`（`main.ts`、`src/publish/stores/tencent-store.ts` 各新增一个真错误）；最终选定 **`5.8.3`**（当时 5.8.x 最新稳定版）：`tsc -noEmit` 通过、`vitest` 97 个测试全绿、`base-renderer.ts` 与 `exporter.ts` 上全部 `no-unsafe-*` 清零。不要不加验证就跳到最新版——先跑一遍完整 `tsc -noEmit` 确认没有新增的真错误。

  > **⚠️ 2026-07-09 更正——上面这条 `base-renderer.ts:570-593` 的"已修复"结论是错的，`exporter.ts` 那部分才对。** 审核机器人在 2026-07-09 对 commit `03e1c62`（已确认包含 `typescript@5.8.3`，且 `base-renderer.ts` 自 2026-07-07 起源码逐字节未变）重新给出了**一模一样**的 `base-renderer.ts:570-593` 报告。用同样方法（`typescript-eslint@7` + 候选 `typescript` 版本）复测显示 0 问题——但这只是因为验证用的 `typescript-eslint@7` 太旧，根本没能力复现。换成 `typescript-eslint@8.63.0`（当前 `latest`，其 `package.json` 声明的 peerDependency 是 `"typescript": ">=4.8.4 <6.1.0"`——**latest 版本的 typescript-eslint 本身就不支持 TS ≥6.1**，所以真实机器人不可能跑比 6.1 更新的 TS）配合它能吃的最高版本 `typescript@6.0.3` 重新验证，**完整复现了全部 15 条报告**，且 `tsc --noEmit` 在 6.0.3 下直接给出未被吞掉的真错误：`TS2339: Property 'sort' does not exist on type '"" | {...}'`。根因和 `exporter.ts` 那次完全不同、与 TS 版本本身无关——是源码里 `(viewName && views.find(...)) ?? views[0] ?? {}` 这行的真实类型缺陷（详见下面"`x && y ?? z` 空字符串陷阱"一节），TS 6.x 的控制流分析比 5.8.3 更早捕捉到了它。**教训：只用一个固定的 `typescript-eslint` 大版本本地复测，永远只能验证那个大版本能看到的问题——必须同时覆盖 `typescript-eslint` 自身的大版本，尤其是它当前 `latest` 版本，才能贴近机器人可能正在用的工具链。**

  **如何独立验证**（本仓库没有装 eslint/typescript-eslint，需要临时搭一套，装在项目目录外——例如本会话的 scratchpad——不落进 `package.json`/lockfile）：

  **⚠️ 关键footgun，务必避免**：不要把 `eslint`/`typescript-eslint` 装在一个共享沙盒目录里，然后只靠反复 `ln -sf` 目标仓库的 `node_modules/typescript` 去切换候选版本。Node 的 `require('typescript')` 是从**发起 require 调用的模块自身所在目录**开始向上找 `node_modules`，而 `@typescript-eslint/*`/`ts-api-utils` 这些包物理上装在沙盒目录里，所以它们解析到的永远是**沙盒自己的** `node_modules/typescript`，跟目标仓库里那个符号链接完全无关——改了也白改，会让每一版的复测结果看起来一样（这正是本节 2026-07-09 那次更正之所以发生的直接原因：第一次复测就是这样被误导成"0 问题"的）。同理，若沙盒目录本身又被别的 `npm install typescript@X` 命令原地复用（哪怕只是为了"顺便探一下这个版本装不装得上"），也会静默地把它内部实际生效的 typescript 版本换掉。

  **正确做法：每个候选版本起一个完全独立、自包含的临时项目**，`eslint`/`typescript-eslint`/`typescript` 三者在同一次 `npm install` 里一起装进同一个 `node_modules`，让内部 `require('typescript')` 就地解析到你要测的那个版本；再从仓库目录用**绝对路径**调用这个独立项目里的 `eslint` 可执行文件：

  ```bash
  SANDBOX=/path/to/scratchpad   # 装在项目外，不要用目标仓库自己的 node_modules
  for V in <候选版本1> <候选版本2>; do
    D="$SANDBOX/verify-$V"
    mkdir -p "$D" && cd "$D" && npm init -y >/dev/null
    npm install eslint@8 typescript-eslint@7 "typescript@$V"   # 三者一起装，同一 node_modules 内自洽
  done

  # 对每个候选版本：在仓库目录写一份指向该沙盒 typescript-eslint 的 flat config，
  # 再用该沙盒的 eslint 可执行文件（绝对路径）跑，全程不碰仓库自己的 node_modules
  cd <repo>
  D="$SANDBOX/verify-<候选版本>"
  cat > eslint.config.mjs <<EOF
  import tseslint from '$D/node_modules/typescript-eslint/dist/index.js';
  export default tseslint.config(...tseslint.configs.recommendedTypeChecked, {
    languageOptions: { parserOptions: { project: '<repo>/tsconfig.json', tsconfigRootDir: '<repo>' } },
  });
  EOF
  node "$D/node_modules/.bin/eslint" <被报告的文件>
  rm eslint.config.mjs   # 验证完清理，不要提交
  ```

  **候选版本怎么选，才算"贴近机器人可能在用的版本"**：不要只测报告发生时仓库锁定的那个版本，也要测 `typescript-eslint` 自己当前 `latest` 版本 + 它 peerDependency 允许的最高 `typescript`（`npm view typescript-eslint@latest peerDependencies` 能查到范围，比如 `>=4.8.4 <6.1.0`，那就测 `<6.1.0` 里能装到的最新稳定版）——这是唯一能验证"最新维护的机器人工具链"的方式，因为 `typescript-eslint` 的类型消解行为在大版本之间可能有实质差异（详见上面 2026-07-09 更正）。同时用同一版本跑一次 `tsc --noEmit`（不经 `skipLibCheck` 吞掉的那种，即直接跑，不加 `-p` 覆盖），如果冒出没被 `no-unsafe-*` 提示说明白的具体 `TSxxxx` 错误，那才是能一眼定位根因的地方，比对着 eslint 的"error typed value"猜要快得多。

**⚠️ 这个根因不止 `.includes()` 一个方法——任何比 `tsconfig.json` 声明的 `lib` 更新的内置方法都会中招**（实测案例，**obsidian-image-cluster**，2026-07-10）：该仓库 `tsconfig.json` 原先写着 `"lib": ["DOM", "ES5", "ES6", "ES7"]`（ES7 = ES2016，只到 `Array.prototype.includes`）。审核报告命中了两处完全不同的方法：

- `src/core/dom.ts:6-7` —— `Object.entries(props).forEach(...)`。`Object.entries` 是 **ES2017**（`lib.es2017.object.d.ts`），比 `ES7` 还高一版，于是 `Object.entries(...)` 本身就是 `error` type：对它调用 `.forEach` 触发 `no-unsafe-call`，解构出的 `[key, value]` 触发 `no-unsafe-member-access`，传给 `el.style.setProperty(key, value)`（形参是具名 `string`）触发 `no-unsafe-argument`。
- `src/render/layout.ts:182-186` —— `visibleRows.flat()`。`.flat()` 是 **ES2019**（`lib.es2019.array.d.ts`），同样只在使用处产生一整串 `no-unsafe-call`/`no-unsafe-member-access`/`no-unsafe-assignment`（`flattenedVisible` 被后续代码 `.slice()`、下标访问、塞进 `Set` 全部连带報错）。

两处根因与 `.includes()` 完全相同（`@types/node` 的 `/// <reference lib="es2020" />` 在本地 `tsc` 里悄悄把 `lib.es2017.*`/`lib.es2019.*` 一并拉了进来，`pnpm build` 因此常年干净），只是命中的具体方法不同——**这证明判断标准要看的是"项目实际用到的每一个内置 API 各自需要哪个 lib 版本"，而不是死记"`.includes()` 需要 ES2016"这一条**。修复同样是把 `tsconfig.json` 的 `"lib"` 提到覆盖所有实际用法的最低版本（这里 `Object.entries` 需要 ES2017、`.flat()` 需要 ES2019，两者取高——`"lib": ["DOM", "ES2019"]`，级联包含 ES2015~ES2019，不必逐条列出 ES5/ES6/ES7），并用上面同款「物理移除 `@types/node`、重跑 `tsc -noEmit`」的方法验证修复后不再依赖隐藏兜底：

```bash
# 排查：不要只 grep .includes(，把所有 ES2016 之后新增的常用内置方法一并扫一遍
grep -rEn '\.includes\(|\.flat\(|\.flatMap\(|Object\.(entries|values|fromEntries|getOwnPropertyDescriptors)|Promise\.allSettled|\.padStart\(|\.padEnd\(|\.trimStart\(|\.trimEnd\(|\.matchAll\(|\.replaceAll\(' src/ main.ts

# 验证：物理移除 @types/node，确认新 lib 配置本身（而非隐藏 reference）就能让 tsc 通过
mv node_modules/@types/node node_modules/@types/node.disabled
npx tsc -noEmit -skipLibCheck
mv node_modules/@types/node.disabled node_modules/@types/node
```

### `x && y.find(...) ?? fallback` 空字符串陷阱 — 看似 `error typed value`，实为真实类型缺陷

**这条容易被误判为"又是某个第三方库 `.d.ts` 解析失败"（参考上面 `error` vs `any` 那节），但根因完全不同、和 TS 版本本身无关，是源码里一个真实的类型窟窿。** `??` 只在左侧是 `null`/`undefined` 时才取右侧兜底值，对其他假值（空字符串 `""`、`0`、`false`）一概放行。当左侧表达式是 `x && expr`（而不是 `x !== undefined ? expr : undefined`）、且 `x` 的类型包含 `string`/`number` 等有假值的原始类型时，`x` 为假值那一分支会把 `x` 本身（而非 `undefined`）短路返回——这个假值会原样穿过后面的 `?? fallback`，变成整个表达式类型里一个多余的字面量分支：

```typescript
// ❌ — obsidian-link-trace src/render/base-renderer.ts:553（2026-07-09 实测）
// viewName?: string；当 viewName === "" 时，`viewName && views.find(...)` 短路返回 ""，
// 而 "" 不是 null/undefined，不会被 ?? 拦下，于是 view 的类型里混进了字面量 ""
const view = (viewName && views.find(v => v.name === viewName)) ?? views[0] ?? {};
// 之后所有 view.sort / view.limit / view.filters 都要在 "" 这个分支上找属性，自然找不到

// ✅ — 用显式的 undefined 判断代替 JS 假值短路，不给假值字面量留混入类型的空间
const view = (viewName !== undefined ? views.find(v => v.name === viewName) : undefined) ?? views[0] ?? {};
```

**为什么本地 `tsc -noEmit` 常年干净、只有升级到更新的 TypeScript（或换用更新的 `typescript-eslint`）才会暴露**：这是 TS 编译器控制流分析精度的问题，不是某个开关；较新版本对 `&&` 短路结果的字面量类型跟踪更精确，会把这类假值分支纳入推断，从而在使用处报出实打实的 `TS2339: Property 'xxx' does not exist on type '"" | {...}'`；较旧版本可能把整个 `&&` 表达式粗粒度地推宽，不出现这个字面量分支，因而查不出来。`@typescript-eslint`（在配了 `skipLibCheck` 等吞掉第三方 `.d.ts` 解析失败的情况下）会把这类真错误也一并归入同一批 "error typed value" 的 `no-unsafe-*` 报告里，和"某个依赖的 `.d.ts` 在老 TS 下解析失败"这种纯工具链问题**长得一模一样**，必须靠 `tsc --noEmit`（不加 `skipLibCheck` 遮蔽，或直接换更新的 TS 版本跑）看到具体 `TSxxxx` 错误信息，才能分清是源码真 bug 还是纯粹的版本兼容问题——处理方式截然不同：真 bug 要改源码逻辑，版本兼容问题可能只需换个 TS 版本或忽略。

**排查方法**：`grep -rn '&&.*?? \|&&.*??' src/*.ts` 找出 `x && y ?? z` 形状的表达式，重点看 `x` 的类型是否包含 `string`/`number`（有假值的原始类型）——`boolean`/对象/`Type | undefined` 这类没有"非 null/undefined 假值"的类型不受影响，无需改。

**非空断言 `x!` 的同名陷阱（lint 与本地 tsc 不同步）**：本仓库 `strictNullChecks: true`，本地 `tsc` 下 `WorkspaceLeaf | null` 的 `x!` 是必要的；但 Obsidian 提交机器人的 type-check **不开 strictNullChecks**，`null` 不被跟踪，`x` 已是非空类型，于是 `x!` 被判为多余而报同一条 `no-unnecessary-type-assertion`。

陷阱在于：把布尔结果存进**单独的变量**（如 `canReuse`）后，TS 无法借它收窄原变量，所以才被迫写 `x!`。**正确修复不是删 `!`（删了本地严格 tsc 会报 null）**，而是在使用处把判空重新并入条件，让控制流自行收窄——既满足机器人（无断言），又满足本地严格 tsc（已收窄）：

```typescript
// ❌ — canReuse 已隐含 active 非空，但 TS 无法据此收窄 active，只能写 active!
const active = this.app.workspace.getMostRecentLeaf();   // WorkspaceLeaf | null
const canReuse = !!active && !(active as LeafInternal).view?.file;
const leaf = canReuse ? active! : this.app.workspace.getLeaf('tab');

// ✅ — 在三元条件里重并 `&& active`，true 分支 active 被收窄为 WorkspaceLeaf，无需断言
const leaf = canReuse && active ? active : this.app.workspace.getLeaf('tab');
```

> 判断信号：报告里 `no-unnecessary-type-assertion` 指向一个 `x!`，而本地 `tsc -noEmit` 却不报错（甚至删掉 `!` 后本地 tsc 反而报 null）——这就是 lint 与本地 strict 配置不同步，用「重并判空收窄」而非「删断言」修复。

**第三种变体——"receiver accepts the original type" 而非 "does not change the type"**：机器人有时会用不同措辞报同一条规则：`This assertion is unnecessary since the receiver accepts the original type of the expression.` 这个变体不是说表达式自身经控制流收窄后已是目标类型（那是上面两种），而是说**接收这个值的位置**（返回值类型、参数类型、赋值目标的字段类型）本来就能接受断言前的类型，断言完全不影响任何一方——常见于 `as unknown as X` 这类双重转换，用来绕过内部/非官方类型的结构不兼容，但如果 getter 的声明返回类型本身就是 `X | null`，转换根本没必要：

```typescript
// ❌ — src/single-page/GraphSidebarManager.ts:28（同款还出现在 ResponsiveSidebarManager.ts:51）
// getter 的声明返回类型是 WorkspaceSidedock | null，可它接收的正是转换前的值，转换是多余的
private get leftSplit(): WorkspaceSidedock | null {
    return (this.app.workspace.leftSplit as unknown as WorkspaceSidedock) ?? null;
}

// ✅ — 若真的需要绕开官方类型定义的结构差异，直接标注变量类型即可，不必双重转换
private get leftSplit(): WorkspaceSidedock | null {
    const split = this.app.workspace.leftSplit;
    return split ?? null;
}
```

> 判断信号：断言链路是 `as unknown as X`，且断言结果被直接 `return`/传参/赋值给一个类型早已声明为兼容 `X`（或更宽）的位置——先看接收方的类型声明，而不是表达式自身的控制流收窄。

同一变体的另一种常见形态：**泛型函数的形参，`T` 已由*另一个*实参推断确定，于是这个实参上的 `as T` 纯属多余**（实测，obsidian-better-link-display `src/lookup.ts:53`，2026-08-15）。这里容易犹豫"删了会不会退化成宽类型"（那是上面 `closest<E>` 那条陷阱的情形），判据是**看 `T` 是从哪里推断出来的**：若另有一个实参已经把 `T` 钉死，被报告的这个实参只是被*检查*、不参与推断，删掉断言后对象字面量会被 `T` 上下文类型化（`reason: "timeout"` 照样收窄成字面量类型），不会变宽：

```typescript
// ❌ — T 由第一个实参 () => Promise<LookupOutcome> 推出，第二个实参的 as 不改变任何一方
private async withTimeout<T>(task: () => Promise<T>, onTimeout: T): Promise<T>
return this.withTimeout(() => this.request(url), { ok: false, reason: "timeout" } as LookupOutcome);

// ✅
return this.withTimeout(() => this.request(url), { ok: false, reason: "timeout" });
```


**第四种变体——通过一个别名布尔变量间接收窄（TS 4.4+ aliased conditions）**：TypeScript 4.4 起支持"别名条件收窄"——`const disabled = !leaf` 这种把某个变量的判空结果存进一个新的 `const` 布尔变量后，后续 `if (disabled) return;` 依然能让 TS 把 `leaf` 收窄为非空，即使收窄发生在一个稍后才调用的闭包内部（只要 `leaf` 本身从未被重新赋值）。这与上面"存进单独变量后 TS 无法收窄，只能写 `x!`"的陷阱刚好相反——`no-unnecessary-type-assertion` 只是运气好命中了 TS 能处理的这一种别名形式，遇到报告先验证是否属于这一种，而不是默认套用"重并判空"的修法：

```typescript
// ❌ — src/layout/StatusBarMenuManager.ts:258-259（实测，obsidian-minimalism-ui，2026-07-12）
// disabled 是 !leaf 的别名，renderModeSegment 提前 `if (disabled) return;`，
// TS 4.4+ 借助别名条件收窄，从这一行往后（包括下面 for 循环里定义、稍后才触发的
// click/keydown 闭包）leaf 的类型已经是 WorkspaceLeaf，非空断言是多余的
private renderModeSegment(container: HTMLElement, leaf: WorkspaceLeaf | null): void {
    const disabled = !leaf;
    // ...
    if (disabled) return;
    for (const { btn, opt } of buttons) {
        const activate = () => {
            const viewState = leaf!.getViewState();
            void leaf!.setViewState({ ...viewState, state: { ...viewState.state, mode: opt.viewMode, source: opt.source } }).then(updateActiveState);
        };
        // ...
    }
}

// ✅ — 直接删掉两处 `!`，TS 借助别名收窄已经知道 leaf 非空
const viewState = leaf.getViewState();
void leaf.setViewState({ ...viewState, state: { ...viewState.state, mode: opt.viewMode, source: opt.source } }).then(updateActiveState);
```

> 判断信号：断言对象是函数参数（不是通过 `.bind`/闭包捕获的外部变量），且函数体内在断言之前有形如 `if (aliasVar) return;` / `if (!aliasVar) return;` 的提前返回，其中 `aliasVar` 是一个 `const` 且其初始化表达式直接引用了该参数（`!leaf`、`leaf == null` 等）——本地 `tsc -noEmit` 通常也会同步报出这条（因为这是 TS 自身的收窄能力，不像上一种依赖机器人 strictNullChecks 配置差异），可以先跑一次 `tsc -noEmit` 验证是否双边都报错，双边一致时直接删断言即可，不需要额外的判空收窄改写。

### `no-unsafe-call` — 调用一个 `any`/`error` 类型的值

机器人原文：`Unsafe call of an 'error' or 'any' typed value.` 触发于把一个未标注类型（因而退化为 `any`）的值当函数调用，常见来源与 [`.bind()`/`.call()`/`.apply()` 一节](#bind--call--apply-返回-any--开-strictbindcallapply) 同源（未开 `strictBindCallApply` 导致整条链是 `any`），也可能来自 `catch` 变量、第三方无类型 SDK、或 `JSON.parse()`/`parseYaml()` 结果直接调用。修复思路一致：先把值的类型钉死，而不是逐处加 `eslint-disable`。

```typescript
// ❌ — origBack 是 any（bind 未开 strictBindCallApply），调用触发 no-unsafe-call
const origBack = history.back.bind(history);
origBack();

// ✅ — 见上面 strictBindCallApply 一节，根因修复后 origBack 自动获得正确签名
```

```bash
grep -rn "no-unsafe-call" # 来自 eslint 输出，定位实际触发行
```

### `.replace()` 回调参数未标注类型

`.replace()` 回调参数默认为 `any`，会触发 `@typescript-eslint/no-unsafe-argument`：

```typescript
// ❌
content.replace(/pattern/, (match, group1, group2) => {
    return fn(group1); // group1 is any → error
});

// ✅ 显式标注回调参数类型
content.replace(/pattern/, (match: string, group1: string, group2: string | undefined) => {
    return fn(group1);
});
```

可选捕获组类型写 `string | undefined`，标注后可去掉多余的 `as string` 断言。

### `any` 值在字符串上下文中 → `[object Object]`

变量类型为 `any` 时，用于字符串模板或拼接会变成 `[object Object]`，触发 `@typescript-eslint/no-base-to-string`：

```typescript
// ❌ — val is any, will stringify to "[object Object]"
const result = `${val}`;
const path = basePath + val;

// ✅ 先收窄类型或显式转换
const result = `${val as string}`;
const path = basePath + String(val);
```

grep 定位：

```bash
# 找出 any 类型变量被用于模板字符串的嫌疑位置
grep -n 'getFieldRaw\|fm\[' src/*.ts | grep '\${'
```

### 未使用变量：加 `_` 前缀

函数参数或变量声明后未使用，ESLint 规则要求未使用变量名以 `_` 开头：

```typescript
// ❌
function render(file: TFile, config: Config) { /* config never used */ }

// ✅
function render(file: TFile, _config: Config) { }
```

**未使用的 import** 是不同情况——报告原文同样是 `'x' is defined but never used`，但 `_` 前缀对 import 绑定不适用（重命名绑定要求同步改掉所有引用，等于手动做死代码删除的反面）。确认该导出确实在本文件无引用后，直接删掉这一个 import 名字，而不是改名：

```typescript
// ❌ — normalizeAppGridUrl exported from appGridScan but never called in this file
import { buildAppGridInsertion, normalizeAppGridUrl, parseAppGridConfig } from "./appGridScan";

// ✅
import { buildAppGridInsertion, parseAppGridConfig } from "./appGridScan";
```

删除前用 `grep -rn "normalizeAppGridUrl" src/` 确认该符号在别的文件里仍有用到（比如被另一个模块导入使用）——只删这一处未使用的 import，不要动它的定义或其他文件里的合法引用。

```bash
grep -n "is defined but never used" # 来自 eslint 输出
```

### ESLint 指令注释须带说明

`// eslint-disable` 系列注释若无描述文字，触发 `eslint-comments/require-description`：

```typescript
// ❌
// eslint-disable-next-line @typescript-eslint/no-explicit-any

// ✅
// eslint-disable-next-line @typescript-eslint/no-explicit-any -- ali-oss SDK has no type declarations
```

```bash
grep -n 'eslint-disable' src/*.ts | grep -v ' -- '
```

### 正则表达式问题

**控制字符**：正则中直接写 `\x00` 等控制字符，触发 `no-control-regex`。改用 Unicode 转义或重构逻辑：

```typescript
// ❌
const re = /[\x00-\x1f]/;

// ✅
const re = /[\u0000-\u001f]/;
```

**多余转义**：`\S` 在字符类 `[...]` 外部是合法的，但某些位置会触发 `no-useless-escape`。去掉多余的反斜杠：

```typescript
// ❌  \. inside a character class is just a literal dot — \. → .
const re = /[\.\s]/;

// ✅
const re = /[.\s]/;
```

**模板字符串里的正则**：在 TypeScript template literal 里嵌入 HTML/JS 脚本时，`\S` 会被模板字符串引擎求值为 `S`（backslash 被忽略），导致生成错误的正则。ESLint 同样报 `no-useless-escape`。必须写成 `\\S` 才能在输出字符串里保留 `\S`：

```typescript
// ❌ — \S in template literal → becomes S in output, regex broken
const script = `var m = code.className.match(/language-(\S+)/);`;

// ✅ — \\S in template literal → becomes \S in output
const script = `var m = code.className.match(/language-(\\S+)/);`;
```

```bash
grep -n 'eslint.*no-useless-escape\|\\\\[^nrtuvxuU0-9$()*+.?[\]^{|}\\]' src/*.ts
```

### `no-unsafe-return` — 从有类型函数返回 `any`

函数声明了返回类型（如 `: string`），但实际返回了 `any` 值，触发 `no-unsafe-return`。常见来源：Obsidian `FrontMatterCache` 中的 frontmatter 属性（类型为 `Record<string, any>`）。

```typescript
// ❌ — frontmatter.share_link 是 any，返回 any 但函数声明返回 string
private getShareLink(file: TFile): string {
    return this.app.metadataCache.getFileCache(file)?.frontmatter?.share_link ?? "";
}

// ✅ — 加 as string | undefined 收窄类型
private getShareLink(file: TFile): string {
    return (this.app.metadataCache.getFileCache(file)?.frontmatter?.["share_link"] as string | undefined) ?? "";
}
```

```bash
grep -n 'frontmatter\?\.' src/*.ts main.ts
```

### `no-floating-promises` — Promise 未被处理

onClick / 事件回调是同步函数，无法 `await`。在其中调用 `async` 方法时，返回的 Promise 被"悬空"，触发 `Promises must be awaited...`。用 `void` 显式标记 fire-and-forget：

```typescript
// ❌ — doPublish returns Promise, unhandled inside sync onClick
menu.addItem(item => item.onClick(() => {
    this.doPublish(file, subNotes);
}));

// ✅
menu.addItem(item => item.onClick(() => {
    void this.doPublish(file, subNotes);
}));
```

```bash
grep -n 'this\.\(doPublish\|doUnpublish\|exportFile\|saveSettings\)(' src/*.ts main.ts
```

### `FrontMatterCache` 的 `any` 属性

Obsidian 的 `FrontMatterCache` 扩展 `Record<string, any>`，所有动态字段均为 `any`。访问这些字段会触发 `no-unsafe-member-access` / `no-unsafe-assignment`。

```typescript
// ❌ — frontmatter.share_link 是 any
const shareLink = cache?.frontmatter?.share_link ?? "";

// ✅ — 用 as 收窄
const shareLink = (cache?.frontmatter?.["share_link"] as string | undefined) ?? "";

// ✅ — processFrontMatter 回调里：显式标注参数类型
await fileManager.processFrontMatter(file, (fm: Record<string, unknown>) => {
    fm["share_link"] = url;
    delete fm["share_link"];
});
```

### 第三方 SDK 无类型声明 → 创建 `.d.ts`

`ali-oss` 等没有官方 TypeScript 声明的 SDK，导致 `new OSS(...)` 及所有方法调用被标记 `no-unsafe-construction` / `no-unsafe-call` / `no-unsafe-member-access`。根治方法：在 `src/` 下建一个最小 `.d.ts` 声明文件，只声明实际用到的接口和方法：

```typescript
// src/ali-oss.d.ts
declare module "ali-oss" {
  interface OSSListParams {
    prefix?: string;
    delimiter?: string;
    "max-keys"?: number;
    marker?: string;
  }
  interface OSSListResult {
    objects?: { name: string }[];
    prefixes?: string[];
    isTruncated?: boolean;
    nextMarker?: string;
  }
  interface OSSPutOptions { mime?: string; }
  interface OSSDeleteMultiOptions { quiet?: boolean; }
  interface OSSConfig {
    region: string;
    accessKeyId: string;
    accessKeySecret: string;
    bucket: string;
    authorizationV4?: boolean;
  }
  class OSS {
    constructor(config: OSSConfig);
    list(params: OSSListParams, options?: object): Promise<OSSListResult>;
    put(key: string, data: Buffer, options?: OSSPutOptions): Promise<void>;
    delete(key: string): Promise<void>;
    deleteMulti(keys: string[], options?: OSSDeleteMultiOptions): Promise<void>;
  }
  export = OSS;
}
```

一旦类型声明存在，所有 `no-unsafe-*` 警告消失，还可以删除手写的 `as { objects?: ... }` 强制类型转换（它们变成 `no-unnecessary-type-assertion`）。

**catch 变量**：tsconfig 不含 `useUnknownInCatchVariables` 时，`catch(e)` 的 `e` 是 `any`，触发 "Unsafe assignment of an error typed value"。对每个 catch 加 `: unknown`：

```typescript
// ❌
} catch (err) { console.warn(err); }

// ✅
} catch (err: unknown) { console.warn(err); }
```

### 废弃 API：`PluginSettingTab.display()` → 迁移到 `getSettingDefinitions()`

机器人原文：`` `display` is deprecated. Since 1.13.0. Use {@link getSettingDefinitions} instead. `` Obsidian 1.13.0 把 Settings Tab 的渲染方式从"整体重写 `display()`"改成声明式的 `getSettingDefinitions()`：只要这个方法返回非空数组，框架就完全接管渲染，`display()` 根本不会被调用（不是"两者都执行、`display()` 优先级更低"）。**先确认这条报告不是陈旧的**——若仓库的 `node_modules/obsidian/obsidian.d.ts` 里 `getSettingDefinitions` 还不存在（`grep -n getSettingDefinitions node_modules/obsidian/obsidian.d.ts`），说明本地 `obsidian` 类型包版本早于 1.13.0，先升级 `obsidian` devDependency 再动手。

**不要用 `eslint-disable` 压制了事——完整迁移的风险比看起来小，因为可以把原有的命令式 `Setting` 构建代码几乎原样保留**，只是换了一层入口。关键决策：

**用 `render`-type definition 包住原有代码，不要用 `control`+`key` 的声明式绑定。** `SettingDefinition` 联合里有两种形态：`control`-type（`{ control: { type: 'toggle', key: 'foo', ... } }`，框架通过 `getControlValue(key)`/`setControlValue(key, value)` 读写，需要新增一层按 key 分发的存取逻辑）和 `render`-type（`{ render: (setting: Setting) => {...} }`，框架建好一个空的 `Setting` 行传进来，剩下的完全由你摆布）。已有的 `new Setting(el).setName(...).addToggle(...).onChange(...)` 代码几乎可以逐字保留在 `render` 回调里，只是把 `new Setting(container)` 换成回调参数 `setting`——风险和改动量都远小于为每个字段重新设计 `key`/`getControlValue`/`setControlValue`：

```typescript
// 之前（imperative display()）
new Setting(sectionEl)
    .setName(t('hideTabBar'))
    .addToggle(toggle => toggle
        .setValue(this.plugin.settings.hideTabBar)
        .onChange(async v => {
            this.plugin.settings.hideTabBar = v;
            await this.plugin.saveSettings();
        }));

// 之后（getSettingDefinitions，render-type，逻辑逐字保留）
{
    name: t('hideTabBar'),
    render: (setting) => {
        setting.addToggle(toggle => toggle
            .setValue(this.plugin.settings.hideTabBar)
            .onChange(async v => {
                this.plugin.settings.hideTabBar = v;
                await this.plugin.saveSettings();
            }));
    },
},
```

**`render` 回调的返回类型是 `void | (() => void)`，不是 `Setting`**——`Setting.addToggle`/`addDropdown`/`addText` 都返回 `this`，箭头函数写成表达式体（`render: (setting) => setting.addToggle(...)`）会把这个 `Setting` 当成返回值，触发 `Type '(setting: Setting) => Setting' is not assignable to type 'undefined'`。必须用花括号语句体丢弃返回值：

```typescript
// ❌ — 表达式体隐式返回 Setting
render: (setting) => setting.addToggle(toggle => toggle.setValue(x).onChange(fn)),

// ✅ — 语句体返回 void
render: (setting) => { setting.addToggle(toggle => toggle.setValue(x).onChange(fn)); },
```

**分组（`SettingDefinitionGroup`）的 `items` 字段类型是 `SettingGroupItem[]`（`SettingDefinition | SettingDefinitionPage`），不是顶层 `getSettingDefinitions()` 返回的 `SettingDefinitionItem[]`（多了 `Group`/`List`）。** 把一个类型为 `SettingDefinitionItem[]` 的数组传给某个 group 的 `items`，TS 会报出一个和实际错误位置错位的confusing 提示（例如 `Type '"group"' is not assignable to type '"page"'`，看起来在怪你外层的 `type: 'group'` 字面量，实际是 `items` 类型不兼容导致 TS 退化去匹配 `SettingDefinitionPage`）。收到这类"类型字面量对不上"的报错时，先检查传入 `items` 的数组元素类型是不是被错误地声明为顶层联合，而不是死抠报错指向的那一行：

```typescript
// ❌ — items 参数类型太宽，包含 group/list，group.items 期望更窄的 SettingGroupItem[]
private section(key: string, title: string, items: SettingDefinitionItem[]): SettingDefinitionItem[] {
    return [
        { name: title, render: renderHeading },
        { type: 'group', visible: () => !collapsed, items },   // TS2322，报错指向别处
    ];
}

// ✅
private section(key: string, title: string, items: SettingGroupItem[]): SettingDefinitionItem[] { ... }
```

**没有原生的"可折叠分组标题"控件——自定义标题行 + 分组的 `visible` 谓词组合实现，同时可以删掉旧的 CSS 相邻兄弟选择器 hack。** `SettingDefinitionGroup.heading` 只是静态文本，没有点击折叠的能力。做法：分组标题做成一个独立的、位于 group **外面**的 `render`-type item（在其 `render` 回调里 `setting.settingEl.empty()` 后完全自定义 DOM，复刻原来手写的可折叠标题），点击时只切换 `collapsedSections[key]` 并调用 `this.refreshDomState()`；真正的显隐交给紧跟其后的 `{ type: 'group', visible: () => !collapsed, items }`——框架原生处理隐藏（大概率是直接给整个 group 的容器元素设置隐藏），不再需要旧版那种 `.heading-collapsed + .content { display: none; }` 的 CSS 相邻兄弟选择器技巧，可以删掉：

```typescript
private renderSectionHeading(key: string, title: string) {
    return (setting: Setting): void => {
        const headingEl = setting.settingEl;
        headingEl.empty();
        headingEl.className = 'setting-item setting-item-heading my-collapsible-heading'
            + (this.isCollapsed(key) ? ' is-collapsed' : '');
        // ...手动建 nameEl / arrow span，和迁移前一致...
        headingEl.addEventListener('click', () => {
            const nowCollapsed = !this.isCollapsed(key);
            this.plugin.settings.collapsedSections[key] = nowCollapsed;
            headingEl.toggleClass('is-collapsed', nowCollapsed);
            void this.plugin.saveSettings();
            this.refreshDomState();   // 只重估 visible 谓词，不整体重渲染
        });
    };
}
```

**两种重渲染入口，按改动性质选：** `this.update()` 重新调用 `getSettingDefinitions()` 并整体重渲染——用在会改变*渲染文本本身*的场景（如切换语言后 `name`/`desc` 都是求值一次的静态字符串，不会自动跟着语言状态变化，必须整体重建）。`this.refreshDomState()` 只重估每个 definition 的 `visible`/`disabled` 谓词函数并原地切 CSS，不重建 DOM——用在结构不变、只是某一行该不该显示发生变化的场景（分组折叠、"高级选项开启后才显示的附属字段"），比整体 `update()` 更轻量，是这两种场景下的官方推荐用法（文档原文："Cheap: toggles CSS state in place, no re-render."）。

**别忘了同步升 `manifest.json` 的 `minAppVersion` 到 `"1.13.0"`**（`getSettingDefinitions`/`refreshDomState`/`update` 均标注 `@since 1.13.0`）——否则会立刻触发本文档"`no-unsupported-api`"一节描述的问题。这是一次真正的破坏性提升（老版本 Obsidian 用户将无法更新到这个版本的插件），迁移前应与维护者确认可以接受。

**⚠️ 真实案例：这个破坏性提升曾经被直接 revert（obsidian-minimalism-ui，commit `f56f8f0`，2026-07-14）。** 该仓库 `minAppVersion` 长期锁定在 `1.7.2`，早先按上面的做法把 `display()` 整个删掉、只留 `getSettingDefinitions()`，随手把 `minAppVersion`/`versions.json` 一并升到 `1.13.0`——维护者随后判断"为了搜索索引这一个次要收益，切断所有 1.7.2–1.12.x 用户的更新通道不划算"，整体回退回纯 `display()` 实现。

**⚠️⚠️ 2026-07-22 更正——下面这条"双路径方案"本身是错的，已被官方审核实测证伪，不要再用。** 2026-07-21 曾在这里记录过一个"不破坏兼容性"的双路径方案：`display()` 和 `getSettingDefinitions()` 并存，把所有 `this.update()`/`this.refreshDomState()` 调用都包一层 `typeof this.update === 'function'` 运行时判断，理由是"老版本运行时这两个方法不存在，加判断后不会抛错，minAppVersion 就不用动"。这套方案提交审核后，**被 Obsidian 官方审核直接判定为 `obsidianmd/no-unsupported-api` Error**（"Uses Obsidian APIs newer than the declared minAppVersion"），精确命中了那个 `typeof this.update === 'function'` 判断内部的 `this.update()` 调用行、以及 `renderSectionHeading` 里裸调用的 `this.refreshDomState()` 行——**审核机器人做的是纯静态分析：只看"这个调用点引用的 API 签名版本是否 ≤ minAppVersion"，完全不理解、也不执行任何运行时判断逻辑**。换句话说，`no-unsupported-api` 检查的是*文本里出现的调用点*，不是*运行时是否可达*——哪怕这行代码被包在一个只在新版本才会为真的 `if` 分支里，只要它在源码里以文本形式引用了 `SettingTab.update`/`refreshDomState`，机器人就会报错，且是 Error 级别（直接拒绝发布），不是 Warning。

**正确结论：只要 `minAppVersion` 不打算提升到 `1.13.0`，`getSettingDefinitions()`/`this.update()`/`this.refreshDomState()` 就不能出现在这个文件的任何代码路径里，无论是否加运行时守卫。** 没有"鱼与熊掌兼得"的中间方案——两条路只能二选一：

- **A. 提升 `minAppVersion` 到 `1.13.0`**，只保留 `getSettingDefinitions()`，删掉 `display()`——这是 obsidian.d.ts 注释和大多数指南默认假设的路径，但代价是切断所有 `1.7.2`–`1.12.x` 用户的更新通道，必须与维护者确认这个代价可接受（该仓库两次判断都是不接受）。
- **B. 完全不实现 `getSettingDefinitions()`，只保留 `display()`。** 审核报告里"does not implement getSettingDefinitions()"这条留作永久性的、不可修的 Warning/Recommendation（不是 Error，不阻断发布）——这是唯一在保持低 `minAppVersion` 前提下能通过审核的选择。`configureX(setting: Setting)` 这种把每个设置项的构建逻辑拆成独立私有方法、被 `display()` 里的 `new Setting(sectionEl)` 调用的写法仍然是好的代码组织，可以保留，只是不要再画蛇添足地为它们外加一层 `getSettingDefinitions()` 声明式包装。

第二个数据点（obsidian-better-link-display，2026-08-15）：`minAppVersion` 是 `1.0.0`，审核同时给出 `does not implement getSettingDefinitions()`（Warning）和 `` `display` is deprecated ``（Recommendation）——当天按方案 B 留着不修，因为这个仓库的整套设计（styles.css 不依赖任何新语法、代码不用任何 1.0.0 之后的 API）就是围绕"尽可能低的 minAppVersion"展开的。**这两条都不是 Error，不阻断发布**；提交前确认报告里它们的级别，别把 Warning 当成必须清零的项而去做破坏性升级。

**⚠️ 2026-08-16 更新——同一个仓库（obsidian-better-link-display）第二天改走了方案 A。** 结论不是"B 是对的、A 是错的"，而是**A/B 是一个产品决策而非技术判断，必须由维护者拍板，不能由代码里的历史痕迹自动推导**：同一个仓库、同一条 Warning、同一个 `minAppVersion 1.0.0` 的设计初衷，隔一天就得到了相反的答案。所以修正上一版这里写的"直接选方案 B"——**正确做法是把两条路的代价摆清楚后问维护者**（A：切断 `1.0.0`–`1.12.x` 用户更新通道，换来设置项进入搜索索引；B：Warning 永久留着，不阻断发布），只有"运行时守卫 + 双路径"那条折中是**技术上**已被实测堵死、不需要问也不能选的。

#### 方案 A 迁移清单（obsidian-better-link-display，2026-08-16 实测走通，`tsc -noEmit` + 32 个单测通过）

除了上面已经写过的 `render`-type 包法、语句体丢弃返回值、`items` 类型收窄之外，这次新踩到/新确认的点：

1. **`display()` 直接删掉即可，不用留空实现。** `SettingTab.display()` 在 1.13 的 `obsidian.d.ts` 里是**非 abstract** 的普通方法（`PluginSettingTab` 本身是 `abstract class`，但没有把 `display` 声明为 abstract），子类不实现照样编译通过。注释原文也确认了语义："Not called when `getSettingDefinitions` returns a non-empty array … Only implement display() as a fallback for plugins that need to support Obsidian versions older than 1.13.0."
2. **`new Setting(el).setName(x).setHeading()` 的等价物是分组，不是某个 "heading" 型 definition。** 原来靠 `setHeading()` 排出来的"标题 + 若干行"的视觉分组，迁移后写成 `{ type: 'group', heading: t('...'), items: [...] }`；`type` 字段是必填的（`'group' | 'list'`）。
3. **`SettingDefinitionBase.name` 是必填 `string`——纯展示性的行也得有名字。** 本仓库有一行是"外观预览"（一个假的书签样本，本身没有任何文案），迁移前它靠 `new Setting(el).setName().setDesc()` + 往 `controlEl` 里塞 DOM 实现。definition 里没有"无名行"这种形态，所以 `name`/`desc` 照填，DOM 构建全部挪进 `render` 回调（`setting.controlEl.createDiv(...)`、`setting.settingEl.addClass(...)` 这些在回调里原样可用，因为拿到的就是框架建好的那个 `Setting` 实例）。
4. **`versions.json` 要补历史条目，光改 `manifest.json` 不够。** `manifest.json` 的 `minAppVersion` 描述的是**下一个**要发的版本，`versions.json` 是"插件版本 → 该版本要求的最低 app 版本"的历史映射表，老版本 Obsidian 靠它回退到最后一个兼容的插件版本。`version-bump.mjs` 是在 `npm version` 时读当时的 manifest 写进去的，所以**必须先确认表里没有缺口再改 manifest**——本仓库 `1.0.2` 已打 tag 发布但 `versions.json` 里漏了这条，如果直接把 manifest 改成 `1.13.0` 就发下一版，1.12 及以下的用户会因为查不到 `1.0.2` 的条目而拿不到最后那个兼容版本：

   ```bash
   # 对照：已发布的 tag 是否都在 versions.json 里有条目
   git tag --list; cat versions.json
   ```

   补完缺口（`"1.0.2": "1.0.0"`）再把 manifest 的 `minAppVersion` 提到 `1.13.0`，下一次 `npm version` 会自动写入新条目。
5. **升完 `minAppVersion` 顺手回收那些"为了兼容老版本而绕路"的写法。** 这类代码通常在注释里自己写明了原因，grep 注释比 grep API 名更容易找全：

   ```bash
   grep -rn "minAppVersion\|only exists from\|@since\|Obsidian 1\.[0-9]" src/ main.ts CLAUDE.md README*.md
   ```

   本仓库命中一处——`button.buttonEl.disabled = true` 外加一句"`ButtonComponent.setDisabled` 只有 1.2.3+ 才有，而 manifest 承诺 1.0.0"的注释，升版后换回 `button.setDisabled(true)` 并删掉注释。同一次 grep 还会把 `README.md` / `README.zh.md` 的 "Requirements: Obsidian 1.0 or later" 和 `CLAUDE.md` 里写死的 minAppVersion 一并翻出来，这些面向用户的版本声明必须跟着改，否则文档会与 manifest 互相矛盾。
6. **语言切换用 `this.update()`，别用 `refreshDomState()`。** 这条上面已经写过原理，实操时容易选错：`name`/`desc` 是**建 definition 那一刻求值一次**的普通字符串，`refreshDomState()` 只重估 `visible`/`disabled` 谓词、不碰文本，切完语言页面会原样不动。



## 依赖项检查（submission bot 扫描）

| 问题 | Fix |
|------|-----|
| `builtin-modules` 包 | 替换为 Node.js 内置：`import { builtinModules } from 'module'` |
| `@typescript-eslint` 旧版本带漏洞传递依赖 | 升级到最新 v5（`5.62.0`）并在 `pnpm-workspace.yaml` 加 overrides |
| `esbuild <=0.24.2` 开发服务器漏洞 | 升级到 `>=0.25.0`；不使用 dev server 时不影响产物，但仍建议升级 |

**判断原则**：漏洞在 `devDependencies` 中且不进入构建产物，不影响插件用户，但审核机器人仍会发出警告——升级可消除警告，建议处理。

```bash
pnpm audit          # 查当前漏洞（比 npm audit 更准确）
npm show @typescript-eslint/eslint-plugin version  # 查最新版
```

升级 `@typescript-eslint` 时注意：
- v8 要求 `typescript >= 4.8.4`，需同步升级
- `@typescript-eslint/eslint-plugin` 和 `@typescript-eslint/parser` 版本必须一致

### pnpm 项目：用 overrides 修复传递依赖漏洞

pnpm v10 的 overrides 需写在 `pnpm-workspace.yaml`（不是 `package.json`）：

```yaml
# pnpm-workspace.yaml
overrides:
  brace-expansion: "^1.1.13"   # GHSA-f886-m6hf-6m8v
  picomatch: "^2.3.2"          # GHSA-3v7f-55p6-f55p, GHSA-c2c7-rcm5-vvqj
  minimatch: "^3.1.4"          # GHSA-23c5-xmqv-rm74 等
  flatted: "^3.4.2"            # GHSA-25h7-pfq9-p65f, GHSA-rf6f-7fwh-wjgh
  js-yaml: "^4.1.1"            # GHSA-mh29-5h37-fv8m
  ajv: "^6.14.0"               # GHSA-2g4f-4pwh-qvx6
```

范围需限定主版本（`^` 而非 `>=`），避免解析到不兼容的新 major（如 `picomatch@4`）。

如果项目同时存在 `package-lock.json`（由 npm 生成），需删除再运行 `pnpm install`，否则 `npm audit` 仍读旧文件。

## manifest.json 检查

| 字段 | 违规 | Fix |
|------|------|-----|
| `description` | 以插件名开头（如 `"Image Cluster helps you..."`) | 直接描述功能，去掉开头的插件名 |
| `description` | 包含 "Obsidian" 一词 | 删除该词——由插件目录的上下文隐含，写出来是多余的 |
| `minAppVersion` | 低于代码中实际使用的 API 所需版本 | 升高 `minAppVersion` 至 API 要求的最低版本 |

**Release tag 严禁加 `v` 前缀**：Obsidian 官方明确要求 GitHub Release 的 tag 必须与
`manifest.json` 的 `version` 字段逐字符相等（`1.0.0`，不是 `v1.0.0`）——这是 Obsidian 侧
"No release matches your manifest version" 报错信息里明确写出的两个可能原因之一（另一个是本文档
上面「Release 不能停在 draft」一节的 draft 问题；报错文案会同时提示这两种可能，出现该报错时两个都要
排查，不要只查一个就下结论）。`/obsidian-plugin-publish` 的 Step 3/4 创建新 tag 时必须始终用裸版本号
（`git tag -a "$NEW_VERSION"`，不要拼接 `v` 前缀）；上面 Step 2 提到的 `v<version>` fallback
**只用于向后查找某个仓库历史遗留的旧 tag 做 changelog diff**，绝不能反过来用它作为新 tag 的命名依据。

```bash
# 核实：当前 manifest.json 版本对应的 tag/release 是否真的裸版本号、且已发布
V=$(node -pe "require('./manifest.json').version")
git tag -l "$V" "v$V"                                    # 应只有裸版本号那一行有输出
gh release view "$V" --json tagName,isDraft --jq '.'     # tagName 应等于 $V，isDraft 应为 false
```

**minAppVersion 兼容性检查：** 代码中用到的 API 必须在 `minAppVersion` 对应的版本中已存在。常见不兼容示例：

| API | 最低要求版本 |
|-----|-------------|
| `FileManager.processFrontMatter` | 1.4.4 |
| `Editor.cm` (CodeMirror 6) | 0.14.0 |
| `Workspace.revealLeaf` 返回 `Promise<void>`（可 await） | 1.7.2 |
| `WorkspaceLeaf.isDeferred` / `loadIfDeferred` | 1.7.2 |
| `Vault.getFileByPath` / `getFolderByPath` | 1.5.7 |

**`obsidianmd/no-unsupported-api`（机器人原文：`Uses Obsidian APIs newer than the declared minAppVersion`）**：这条不是看 API 是否存在，而是看你依赖的**签名版本**。典型陷阱：`revealLeaf` 自古就有，但「返回 `Promise<void>`、可被 `await`」是 1.7.2 随 deferred views 引入的——`minAppVersion < 1.7.2` 时调用就报此条。

```typescript
// ❌ minAppVersion 1.5.7 + revealLeaf 现在返回 Promise<void> → no-unsupported-api（且 no-floating-promises）
this.app.workspace.revealLeaf(leaf);

// ✅ 把 minAppVersion 升到引入该签名的版本，并 await（顺带消除 floating promise）
await this.app.workspace.revealLeaf(leaf);   // manifest.json: "minAppVersion": "1.7.2"
```

> 关键：`no-unsupported-api` **只能靠升 `minAppVersion`（或换用旧版兼容 API）修复**——加 `void`/`await` 只解决 floating-promise，不解决本条。同一行同时报这两条时，先升 minAppVersion，再 await。

```bash
# 找出 processFrontMatter 调用
grep -rn "processFrontMatter" src/
# 对照 manifest.json 中的 minAppVersion
cat manifest.json | grep minAppVersion
```

若 `minAppVersion` 低于 API 所需，需二选一：升高 `minAppVersion`，或改用兼容旧版本的替代实现。

**不要靠上面那张表猜版本——`obsidian.d.ts` 每个成员上方就写着 `@since`，直接查。** 报告只给出文件行号（例如 `src/settings.ts:162`），先看那一行调了哪些 Obsidian API，再逐个在类型定义里读它自己的 `@since`（注意：`@since` 挂在**成员**上，不是类上，同一个类里不同方法版本可以差很多）：

```bash
# 若 node_modules 还没装，先 npm install（obsidian 是 devDependency）
grep -n -B6 "    setDisabled(disabled: boolean): this;" node_modules/obsidian/obsidian.d.ts
# → /** @public @since 1.2.3 */ ——这就是该调用点要求的最低版本
```

**实测案例（obsidian-better-link-display，2026-08-15，Error 级别）**：`manifest.json` 写着 `"minAppVersion": "1.0.0"`，设置页里一句再普通不过的

```typescript
button.setDisabled(true).setButtonText(t("token.testing"));   // ❌ ButtonComponent.setDisabled: @since 1.2.3
```

被判 `no-unsupported-api`。**这类"看起来自古就有"的基础组件方法是本条最常见的来源**——`ButtonComponent.setDisabled`(1.2.3)、`setIcon`/`setTooltip`(1.1.0)、`setDestructive`/`removeDestructive`(1.13.0) 都比多数插件声明的 `minAppVersion` 新。

修法除了升 `minAppVersion`，还有第三条常被忽略的路：**当这个 API 只是某个原生 DOM 属性的薄封装时，直接用底层属性，一行改动、零兼容代价**（`Component.xxxEl` 这类 DOM 引用字段本身是老 API，不受影响）：

```typescript
// ✅ 保持 minAppVersion 1.0.0：setDisabled 内部做的就是这件事
button.buttonEl.disabled = true;
button.setButtonText(t("token.testing"));   // setButtonText 是 @since 0.9.7，可以留
```

> 选择依据：`minAppVersion` 是对用户的兼容性承诺，升它会切断老版本用户的更新通道（同样的权衡见上面 `getSettingDefinitions` 一节）。因此顺序是：① 能用底层 DOM 属性/老签名等价替代 → 替代；② 替代不了（真正的新功能，如 deferred views、声明式设置）→ 升 `minAppVersion` 并同步 `versions.json`，且要跟维护者确认。**改完在代码旁留一句注释说明为什么不用那个更顺手的 API**，否则下一个人会"顺手改回去"。

```json
// ❌
"description": "Image Cluster helps you combine multiple images in your notes."

// ✅
"description": "Combine multiple images together in your notes for a more organized layout."
```

检查方法：`cat manifest.json` 查看 `description` 字段是否以插件 `name` 开头。

```json
// ❌
"description": "A simple and powerful todo plugin for Obsidian."

// ✅
"description": "A simple and powerful todo plugin."
```

检查方法：`cat manifest.json` 查看 `description` 字段是否包含 "Obsidian"（大小写不敏感）——该词在插件目录上下文中是冗余信息。

## Release 工作流检查（GitHub Actions）

**GitHub 建议：** Release 资产（`main.js`、`styles.css`）缺少 artifact attestation，无法密码学验证构建来源。

检查 `.github/workflows/release.yml` 是否包含以下三项：

1. `id-token: write` 和 `attestations: write` 权限
2. `actions/attest-build-provenance@v2` 步骤（在 build 之后、create release 之前）
3. `subject-path` 列出所有发布资产

```yaml
# ✅ 正确写法
jobs:
  build:
    permissions:
      contents: write
      id-token: write
      attestations: write
    steps:
      - name: Build plugin
        run: |
          npm install
          npm run build

      - name: Attest build provenance
        uses: actions/attest-build-provenance@v2
        with:
          subject-path: |
            main.js
            styles.css

      - name: Create release
        ...
```

缺少任一项均视为违规，需补全。

### ⚠️ Release 不能停在 draft —— Obsidian 要求有一个已发布的 release 匹配 manifest.json 版本

**真实故障案例**（obsidian-link-trace，2026-07-09）：`gh release create` 带了 `--draft`，且
`/obsidian-plugin-publish` 流程在推送 tag 后就停止，从不检查/发布这个 release。`manifest.json`
在 `main` 分支上已经写着新版本号并被推送，但对应 tag 的 GitHub Release 一直停在 Draft 状态，
直到有人手动点击 "Publish release"。这个手动步骤很容易被忘记——一旦忘记，Obsidian 的插件安装/
更新检查会报错：

```
No release matches your manifest version
Your manifest.json points at a version that doesn't have a matching GitHub release.
...
Publish a release on GitHub
```

原因：Obsidian 通过 GitHub Releases API 按 tag 精确匹配下载资产，**draft release 对该 API 不可见**
（等同于不存在），而仓库里 `manifest.json` 的版本号早已公开可见——于是出现"manifest 说是 X.Y.Z，
但公网找不到 X.Y.Z 的 release"这个不一致窗口，且如果没人记得手动发布，这个窗口永远不会关闭。

**根治方法**：`gh release create` **不要带 `--draft`**，让 CI 构建成功即为发布成功，不存在需要
人记住的手动步骤：

```bash
# ❌ — 需要人工事后点 Publish，容易忘记，忘记期间 manifest.json 版本对不上任何 release
gh release create "$tag" --title="$tag" --draft main.js manifest.json styles.css

# ✅ — CI 绿灯即代表已发布，manifest.json 推送的那一刻就有对应 release
gh release create "$tag" --title="$tag" main.js manifest.json styles.css
```

若某个项目确实需要发布前人工审核（不想让 CI 直接公开），**不要**简单加 `--draft` 了事——那就是
这个 bug 的根源。改为在 CI 里保留 draft，但在 `/obsidian-plugin-publish` 流程的最后新增一步：
等待该次 push 触发的 workflow run 完成，然后 `gh release edit "$tag" --draft=false` 自动发布，
把"记得手动发布"这件事从人的记忆里挪回自动化流程。

**检查现有仓库是否已经踩坑**：

```bash
# 1. 工作流是否硬编码 --draft
grep -n '\-\-draft' .github/workflows/release.yml

# 2. manifest.json 当前版本是否真的有一个已发布（非 draft）的 release
PREV=$(node -pe "require('./manifest.json').version")
gh release view "$PREV" --json isDraft --jq .isDraft   # true → 立刻发布：gh release edit "$PREV" --draft=false
```

### Populating release notes from the tag message

不同项目的发布机制不统一——有的直接调 `gh release create`，有的用第三方 Action（如
`softprops/action-gh-release`）。为了让"release 自动带上本次更新内容"这件事能跨项目复用，
约定把决策点放在本地打 tag 那一步（见上面 Step 2/3）：notes 的实际内容在本地生成，写进
**annotated tag 的 message**；CI 端只需要把这条 tag message 原样读出来，完全不用关心它最初来自哪个
changelog 文件、标题格式长什么样。

**先探测这个仓库用哪种发布机制，不要假设：**

```bash
grep -rl "gh release create\|action-gh-release" .github/workflows/*.yml
```

**若用 `gh release create`（原生 CLI）**——取出 annotated tag 的正文，写入临时文件后传给
`--notes-file`；如果 tag 没有内容（比如遗留的 lightweight tag），fallback 到 `--generate-notes`
（GitHub 按 commit/PR 自动生成），保证不会因为读不到 notes 而报错或产出空 body：

⚠️ **`actions/checkout` 会吞掉 annotated tag 的 message**：默认配置下它只按 `GITHUB_SHA`（tag
指向的 commit，而非 tag 对象本身的 SHA）做浅拉取，本地生成的 `refs/tags/<tag>` 只是指向该 commit
的轻量引用，annotation 内容随之丢失。此时 `git tag -l --format='%(contents)'` 不会报错、也不会为
空——它会**悄悄**返回 commit 的提交信息（因为轻量 tag 的 `%(contents)` 就是其指向 commit 的
message），于是脚本走进第一个分支，release body 变成一行 `chore(release): bump version to
x.y.z`，而不是真正的 changelog。必须在读取前显式按引用名重新拉取一次 tag 对象，覆盖本地那个轻量引用：

```yaml
- name: Create release
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    tag="${GITHUB_REF#refs/tags/}"

    # 强制按引用名重新拉取真正的 tag 对象（而非 actions/checkout 留下的、
    # 指向 commit 的轻量引用），否则下面的 %(contents) 读到的是 commit message。
    git fetch origin "refs/tags/$tag:refs/tags/$tag" --force

    notes="$(git tag -l --format='%(contents)' "$tag")"

    if [ -n "$notes" ]; then
      echo "$notes" > /tmp/release-notes.md
      gh release create "$tag" --title="$tag" --notes-file /tmp/release-notes.md <assets...>
    else
      gh release create "$tag" --title="$tag" --generate-notes <assets...>
    fi
```

（`<assets...>` 替换为该项目实际发布的文件列表，例如 `main.js manifest.json styles.css`。不要加
`--draft`——见上面「Release 不能停在 draft」一节：draft release 对 Obsidian 的下载/更新检查不可见，
如果没人记得手动发布，会让 `manifest.json` 的公开版本号长期对应不上任何可下载的 release。）

**验证方法**：release 发布后用 `gh release view <tag> --json body` 核对 body 是否等于本地
`git tag -l --format='%(contents)' <tag>` 的输出，而不是仅仅"CI 跑绿了"就当作正确——这个 bug
不会导致 workflow 失败，只会静默产出错误内容。同时用 `gh release view <tag> --json isDraft --jq
.isDraft` 确认结果是 `false`。

⚠️ **若项目就是要保留 draft（人工审核后再发布）**：改 notes 时不要用 `gh release edit
--notes-file` 而不带 `--draft`——`gh release edit` 对未显式传入的 `--draft` 会按 `false` 处理，
而不是"保持原值"，只改 notes 就会把 draft release 意外发布成正式 release（`isDraft` 从 `true`
变 `false`，且发布后再改回 draft 会导致 release 的 URL 变成 GitHub 分配的 `untagged-<hash>`
占位符，虽然 `tag_name` 字段和已上传的 assets 不受影响，仍指向原 tag）。修正 body 时必须显式
带上当前的 draft 状态：`gh release edit <tag> --notes-file <file> --draft=true`。

**若用 `softprops/action-gh-release`**——该 Action 在没有显式设置 `body` / `body_path` /
`generate_release_notes: true` 时，通常会默认用 annotated tag 的 message 作为 release body；只需确认
项目没有把这三个字段中的任何一个写死覆盖掉即可，不一定需要改动。改之前用该 Action 当前锁定的版本号
核实一下这条默认行为仍然成立，而不是直接假设。

**改之前先确认这项工作是否已经做过**——如果 workflow 里已经有类似的 tag-message 读取逻辑，不要重复
添加。

## Checking CSS for browser feature compatibility

**Submission bot warning:** `Unexpected browser feature "X" is only partially supported by Obsidian Y.Z`

The bot runs [Browserslist](https://browsersl.ist/) / [caniuse](https://caniuse.com/) checks against the Electron version bundled in the target `minAppVersion`. A "partially supported" warning means the feature works in some Electron builds but not all, or only behind a flag.

Known partially-supported features in Obsidian 1.6.5 / Electron 27:

| Feature key | Affected CSS properties | Fix |
|-------------|------------------------|-----|
| `css-clip-path` | `clip-path` | Add `-webkit-clip-path` fallback; or avoid on non-critical elements |
| `css-masks` | `mask-image`, `mask`, `mask-size`, `mask-position`, etc. | Add the `-webkit-mask-*` prefixed form immediately before each standard property |
| `multicolumn` | `column-count`, `column-width`, `column-gap`, `column-rule` | Add `-webkit-column-*` prefixed fallbacks |
| `css-boxdecorationbreak` | `box-decoration-break` | Add `-webkit-box-decoration-break` |
| `css-text-indent` | `text-indent` with `each-line`/`hanging` keywords | Use plain `text-indent: <length>` without keywords; or drop |
| `text-decoration` | `text-decoration-thickness`, `text-underline-offset`, `text-decoration-skip-ink`, `text-underline-position` | These newer sub-properties may not render in older Electron; add fallback `text-decoration: underline` and treat sub-properties as progressive enhancement |
| `css-display-contents` | `display: contents` | No prefixed form exists — restructure the layout so no box needs to disappear; see the worked example below |

```bash
# Find affected properties
grep -rn 'clip-path\|mask-image\|mask:\|column-count\|column-width\|box-decoration-break\|text-indent.*each-line\|text-indent.*hanging\|text-decoration-thickness\|text-underline-offset\|text-decoration-skip-ink\|text-underline-position\|display: *contents' styles.css theme/
```

> ⚠️ **The version in the warning text is the bot's own baseline, not your `minAppVersion`.** Observed on **obsidian-image-cluster** (2026-08-15): `manifest.json` declared `"minAppVersion": "1.13.4"`, yet the warning read `only partially supported by Obsidian 1.11.4`. Don't try to "fix" this by raising `minAppVersion` — the bot isn't reading that field for this check, and raising it won't silence the warning. Fix the CSS.

For each hit, either add the vendor-prefixed form immediately before the standard property, or drop it if the visual effect is non-essential:

```css
/* Before */
.my-el { clip-path: inset(0 round 4px); }

/* After */
.my-el {
  -webkit-clip-path: inset(0 round 4px);
  clip-path: inset(0 round 4px);
}
```

### `display: contents` — 没有 vendor prefix，只能改布局结构

`display: contents` 的作用是"让这个元素不生成盒子，把它的子元素直接提升为父级布局的参与者"。
它没有任何 `-webkit-` 前缀形式，`@supports` 降级也没有意义（降级分支要写的就是真正的修复），
所以**唯一的修法是重新安排布局，让没有任何盒子需要消失**。

最常见的触发场景：为了便于整体搬运/清理，插件把一个按钮包在自己的 wrapper `div` 里，然后把
**wrapper**（而不是按钮本身）塞进宿主的 flex 容器；为了让按钮仍然表现得像是该 flex 容器的直接
子项，就给 wrapper 加了 `display: contents`。

**先去读宿主容器的真实 CSS，再决定替换成什么** —— 别凭印象猜它是不是 flex。Obsidian 自己的
`app.css` 可以从当前实际运行的版本里抽出来（注意 `/Applications/Obsidian.app` 里那份往往是
安装器的旧版本，自动更新后真正在跑的是 `~/Library/Application Support/obsidian/obsidian-<ver>.asar`）：

```bash
ls ~/Library/Application\ Support/obsidian/*.asar          # 找到真正在运行的版本
npx --yes @electron/asar extract-file \
  ~/Library/Application\ Support/obsidian/obsidian-1.13.7.asar app.css
grep -n '\.embed-actions' -A 12 app.css                    # 读宿主容器的 display / gap
```

（抽出来的 `app.css` 放在 scratchpad 里，**绝不要落进插件仓库** —— 见上面
「Repo hygiene: do not track Obsidian's CSS in the plugin repo」，一旦被 git 跟踪，
bot 会把这两万行里的所有违规都算到你的提交头上。）

如果宿主是 `display: flex`（Obsidian 1.13 的 `.embed-actions` 就是 `display:flex` + `gap`），
把 wrapper 自己改成 `display: flex` 即可 —— 单个子按钮本身也是 flex 盒子，外面套一层
shrink-to-fit 的 flex 容器，排版结果与"wrapper 不生成盒子"完全一致，间距照样由宿主的 `gap` 统一负责：

```css
/* ❌ — 触发 css-display-contents */
.embed-actions > .plugin-image-setting-outer { display: contents; }

/* ✅ — wrapper 变成一个恰好包住按钮的 flex 子项，视觉零差异 */
.embed-actions > .plugin-image-setting-outer { display: flex; }
```

**改完顺手清理因 `display: contents` 而存在的 JS 绕行代码和注释。** `display: contents` 的元素
`getBoundingClientRect()` 恒为全 0，代码里往往留有"必须取内层按钮而不是 wrapper，否则弹层会被
定位到视口外"这类注释甚至 workaround —— 换成 `display: flex` 后 wrapper 有了真实盒子，这些注释
就是错的。`grep -rn 'display: *contents\|contents' src/` 找一遍引用，把理由改写成当前成立的那个
（例如"wrapper 的盒子边界不等于按钮，弹层要贴按钮下沿所以仍取按钮"），不要留着过期解释。

> 宿主不是 flex 而是普通行内流时，用 `display: inline-flex`：在 flex 容器里它会被 blockify 成
> `flex`，两种上下文都安全。不确定就按上面的方法把宿主 CSS 抽出来看一眼。

## Checking CSS for unknown type selectors

**Submission bot warning:** `Unexpected unknown type selector "X"`

The linter runs with a known set of HTML element names. Custom/non-standard element names (from Electron, MathJax, or other renderers) are flagged as unknown.

Common examples:

| Selector | Source | Fix |
|----------|--------|-----|
| `webview` | Electron `<webview>` element | Prefer targeting by class or parent context; if unavoidable, use an `/* stylelint-disable-next-line selector-type-no-unknown */` comment |
| `mjx-container` | MathJax v3 output element | Same — prefer wrapping class or stylelint disable comment |

```bash
grep -rn 'webview\|mjx-container' styles.css theme/
```

If the selector is intentional (you're styling a known third-party element), suppress per-line with a comment rather than globally:

```css
/* stylelint-disable-next-line selector-type-no-unknown */
webview { display: flex; }
```

## Repo hygiene: do not track Obsidian's CSS in the plugin repo

**Root cause:** The submission bot lints **all CSS files found in the GitHub repository**, not just the files in the release zip. If `app.css` (Obsidian's own compiled CSS, ~20 000 lines) is checked into the plugin repo — even for local development reference — the bot will scan it and report every violation it contains, all attributed to your submission.

**Diagnosis:**

```bash
git ls-files "*.css"       # list all tracked CSS files
wc -l *.css                # any file with thousands of lines is suspicious
```

If `app.css`, `obsidian.css`, or any other large Obsidian-internal CSS file appears in `git ls-files`, it must be untracked.

**Fix:**

```bash
# 1. Add to .gitignore
echo "app.css" >> .gitignore

# 2. Remove from git index (keeps the local file, just stops tracking it)
git rm --cached app.css
```

Then commit. The local `app.css` remains for development but is no longer visible to the bot.

Similarly, any unrelated CSS snippet files that are not part of the plugin's runtime output (`customer-ui.css`, `snippets/*.css`, etc.) should be either gitignored or moved outside the plugin repo — unless they are intentionally distributed as part of the plugin.

## Security & Behavior Review（Submission Bot 行为审查）

Obsidian submission bot 对插件行为按三个级别分类：**Warning**（会受到人工审查）、**Recommendation**（知悉即可）、**Pass**（通过）。

| 级别 | 行为 | 触发模式 | 修复方向 |
|------|------|----------|----------|
| Warning | Direct Filesystem Access | `import * as fs from 'fs'`、`import { readFileSync } from 'fs'` | 对 vault 内路径改用 `app.vault.adapter`（`write`、`writeBinary`、`mkdir`）；对 vault 外路径（用户配置的导出路径等）`fs` 是唯一选项，需在注释中说明意图 |
| Warning | Shell Execution | `import { exec } from 'child_process'`、`require('child_process')` | 无法接受——移除；若确实必要需充分说明 |
| Warning | System Identity Information | `os.hostname()`、`os.userInfo()`、`os.networkInterfaces()`、身份相关环境变量读取 | 可能被用于设备指纹识别，审核会重点关注；直接调用应移除或说明用途——常见根因见下方专节 |
| Recommendation | Vault Enumeration | `vault.getFiles()`、`vault.getMarkdownFiles()`、`vault.getAllLoadedFiles()` | 通常为必要功能（链接解析、搜索）；把循环范围收到实际需要的那一类文件即可，不必移除——详见下方专节 |
| Recommendation | Clipboard Access | `navigator.clipboard.writeText()`、`navigator.clipboard.readText()` | 通常为 UX 功能（复制链接、代码块）；确认只在用户触发时调用，无需移除 |
| Pass | Vault Read | `vault.read()`、`vault.cachedRead()` | 无需处理 |

### 检查命令

```bash
# Direct Filesystem Access
grep -rn "from 'fs'\|from \"fs\"\|require('fs')\|require(\"fs\")" src/ main.ts

# Shell Execution
grep -rn "child_process\|exec(\|spawn(" src/ main.ts

# System Identity Information
grep -rn "os\.hostname\|os\.userInfo\|os\.networkInterfaces" src/ main.ts

# Vault Enumeration
grep -rn "vault\.getFiles\|vault\.getMarkdownFiles\|vault\.getAllLoadedFiles" src/ main.ts

# Clipboard Access
grep -rn "navigator\.clipboard\|Clipboard" src/ main.ts
```

### Warning: System Identity Information 的常见根因与修复

这条通常不是插件自己主动指纹识别用户，而是来自**打包进 bundle 的第三方依赖的 stub/polyfill**。典型场景：某个依赖（如云存储 SDK）的间接依赖在 Node 环境下会调用 `child_process` 或探测真实网络信息，为了避免把 `child_process` 打进 Electron bundle，项目里手写了一个替代 stub —— 如果这个 stub 图省事直接调用 `os.networkInterfaces()` 拿真实本机 IP，就会把"移除 `child_process`"的修复动作又变成了"读取系统身份信息"的新警告。

判断是否可以简化为固定值的关键：**这个字段的返回值在插件实际用到的代码路径里是否真的被读取/比较**。如果只是上游库在未使用的功能分支里把它写进一个从不会被检查的文件名/日志里，返回值本身无关紧要，可以直接硬编码一个占位值，完全不调用任何系统探测 API：

```javascript
// ❌ — stub 为了"更真实"去探测本机网络接口，触发指纹识别警告
var os = require("os");
function localIP() {
  var ifaces = os.networkInterfaces();
  // ...遍历找到第一个非内网 IPv4...
  return ip;
}

// ✅ — 上游代码只把这个值拼进一个从不会被读取的文件名，固定值即可，
// 完全不需要 os 模块
function localIP() {
  return "127.0.0.1";
}
```

```bash
# 定位这类"为规避 child_process 而手写的 stub"文件
grep -rln "os\.networkInterfaces\|os\.hostname\|os\.userInfo" src/
```

修复前务必确认返回值确实不影响功能（读一下上游依赖里这个值的实际用途），并在代码注释里说明"为什么固定值也正确"，而不是简单删除注释了事。

### Recommendation: Vault Enumeration —— 收窄范围，不要为了消除它重写数据格式

这条是 **Recommendation**（知悉即可，不阻断发布），bot 原文只是陈述事实："Enumerates all files in
the vault (`vault.getFiles`) — gives the plugin access to every file path in the vault."
**它不要求你删掉这个调用**，要求的是"范围最小化"。判断顺序：

1. **这个功能真的需要"vault 里有哪些文件"这一信息吗？** 如果只是为了拿某个已知路径的文件，
   改用 `vault.getAbstractFileByPath()` / `getFileByPath()`，枚举就地消失，这是最好的结果。
2. **需要，但只关心某一类文件？** 在循环里第一行就按扩展名/路径前缀过滤掉其余文件，并在注释里
   写明"为什么只需要这一类"。这就是 bot 想看到的"范围最小化"。
3. **需要全量、且没有等价 API？** 保留，在报告里标 `[KEPT: reason]`。

⚠️ **不要用"手动递归遍历 `vault.getRoot().children`"来绕过这条。** 那是同一份数据、同一份权限，
只是躲开了 bot 的 grep 模式 —— 属于欺骗检测器而非修复问题，人工复审一眼就能看穿。

⚠️ **也不要为了消除它去重写缓存/索引的数据格式。** 实测案例（**obsidian-image-cluster**，
2026-08-15）：`pruneOrphanedThumbnailCache` 需要"删掉所有不再对应任何原图的缩略图缓存"，缓存
文件名是 `md5(原图路径@分辨率)` —— **单向不可逆**，无法从缓存文件反推原图路径去逐个校验存在性，
只能反过来枚举原图、算出全部合法缓存名。当时评估过两个"彻底移除枚举"的方案，两个都被否决：

| 方案 | 为什么否决 |
|------|-----------|
| 把 md5 换成可逆编码（base64url of path），从缓存文件名反解原图路径 | 改变缓存命名格式 → 所有存量用户的缓存全部失效重建；且把 vault 路径结构明文暴露在文件名里；长路径还有 255 字节文件名上限风险 |
| 维护一份 `thumbPath → 原图路径` 的持久化索引 | 新增一份必须与真实文件保持同步的状态；索引丢失/未迁移时全部缓存会被误判为孤儿而清空 |

为了一条不阻断发布的 Recommendation 承担"清空全体用户缓存"的迁移风险，是明显划不来的交换。
最终采用方案 2（收窄范围）：只枚举图片文件，笔记和其他附件的路径根本不读。

```typescript
// ❌ — 枚举 vault 全部文件，对每个都算一遍缓存名
for (const file of plugin.app.vault.getFiles()) {
  if (file.path.startsWith(config.THUMBNAIL_PATH)) continue;
  validThumbPaths.add(getThumbPath(file.path));
}

// ✅ — 第一行就把范围收到"只有图片"，并注释清楚漏判的后果是安全的
const IMAGE_EXTENSIONS = new Set(["avif", "bmp", "gif", "jpeg", "jpg", "png", "svg", "webp"]);
for (const file of plugin.app.vault.getFiles()) {
  if (!IMAGE_EXTENSIONS.has(file.extension.toLowerCase())) continue;   // 笔记/其他附件永远没有缩略图
  if (file.path.startsWith(config.THUMBNAIL_PATH)) continue;
  validThumbPaths.add(getThumbPath(file.path));
}
```

**收窄时务必确认"漏判"的后果是安全的**，并把这个论证写进注释 —— 上例中漏掉某种未列出的图片格式，
后果只是它的缓存被当成孤儿删掉、下次渲染重新生成，不丢数据；如果漏判会导致数据丢失，那就不能
用扩展名白名单这种"宁可错杀"的过滤方式。

### Warning: Direct Filesystem Access 的修复决策

```typescript
// ✅ vault 内路径 → 用 Obsidian adapter API，不触发 Warning
import { FileSystemAdapter } from 'obsidian';
const adapter = app.vault.adapter as FileSystemAdapter;
await adapter.write('relative/to/vault.md', content);
await adapter.mkdir('relative/dir');

// ✅ vault 外路径（用户配置的外部导出目录）→ fs 是唯一选项，加注释说明
// fs is required here: exportRoot is a user-configured absolute path outside the vault;
// app.vault.adapter only resolves paths relative to the vault root.
import * as fs from "fs";
fs.writeFileSync(path.join(exportRoot, 'file.html'), content, 'utf8');
fs.mkdirSync(dir, { recursive: true });
```

**判断原则**：如果写入目标是用户选择的 vault 外目录（如"导出到桌面"），`fs` 是合理且必要的，审核人通常会批准——但代码中需有注释说明理由，且不应读取任意路径。

## `/obsidian-plugin-publish [version]` Release Command

Runs the version-bump-and-release workflow for the current Obsidian plugin repo. `version` is
optional — an explicit argument (e.g. `/obsidian-plugin-publish 1.2.0`) sets the new tag exactly;
omitting it auto-increments the patch component of the previous version (`0.1.12` → `0.1.13`).

This command **pushes a tag**, which is a shared, hard-to-reverse action that kicks off CI release
automation — confirm the resolved version number and target commit with the user before the final
push (step 4), per the general git-safety rules: never force-push, never skip hooks, and only push
what this workflow explicitly calls for.

### Step 0 — Preflight: the release must be based on `main`

**Run this before anything else.** The release does not have to be cut from a checkout that is
literally on `main`, but the commit that gets tagged **must already be on `main`** — every later step
is irreversible in a shared way, and a tag pointing at a commit that never reached the default branch
makes `manifest.json`'s public version describe code nobody can see on `main` (untangling that means
deleting a published release and a pushed tag).

```bash
git rev-parse --abbrev-ref HEAD
git fetch origin main --quiet
git rev-list --left-right --count origin/main...HEAD    # "<behind>	<ahead>"
```

Evaluate:

- **Current branch is `main`** — nothing to note, proceed.
- **Current branch is not `main`** (feature branch, worktree branch, detached `HEAD`, …) — **just tell
  the user** which branch they are on and that the release will be cut from it. This is a notice, not
  a blocker: proceed to the next check.
- **`HEAD` is *ahead* of `origin/main`** (the `ahead` count is non-zero) — those commits are not on the
  default branch yet, so tagging here would publish code that never landed on `main`. **Stop and ask
  the user** to land the work on `main` first (the `/merge-main` skill does exactly that for the
  current branch), or to confirm explicitly that they want the tag cut from these un-merged commits
  anyway. Do not decide this yourself.
- **`HEAD` is *behind* `origin/main`** (the `behind` count is non-zero) — the checkout is missing
  commits that are already on `main`; the release would be stale. Tell the user and let them decide
  whether to update first.

Do not work around any of this by switching or merging branches yourself — checking out or merging
`main` on the user's behalf is a separate decision that belongs to them.

### Step 1 — Commit outstanding work (skip if clean)

```bash
git status --porcelain
```

If this is non-empty, ask the user whether to commit it before releasing (do not decide unilaterally
which files to stage — follow the normal commit workflow: review the diff, draft a message summarizing
the "why"). If the user declines, proceed anyway — the tag will simply be cut from the current HEAD,
and the uncommitted work stays out of the release. If the tree is already clean, skip this step
silently.

### Step 2 — Generate a changelog entry since the last release, and capture a notes fragment for the tag

1. Read the **previous** version number out of the root `manifest.json`'s `version` field (or wherever
   that repo's own `CLAUDE.md` says the version lives).
2. Find the matching tag. Tags are commonly bare version numbers (`0.1.12`, not `v0.1.12`) — check that
   form first, then `v<version>` as a fallback for repos that do prefix:
   ```bash
   PREV=$(node -pe "require('./manifest.json').version")
   TAG="$PREV"
   git rev-parse -q --verify "refs/tags/$TAG" >/dev/null || TAG="v$PREV"
   git rev-parse -q --verify "refs/tags/$TAG" >/dev/null || TAG=""   # no matching tag found
   ```
   If no tag matches (e.g. first-ever release), say so and fall back to the full log (`git log
   --oneline`) rather than silently guessing a range.
3. **Detect the changelog file, if any — don't assume filename or heading style.** This skill runs
   against many different plugin repos, each with its own (or no) convention. Check the repo root for
   `version.md`, then `CHANGELOG.md`, then `CHANGES.md`. If one exists, read its most recent entry to
   learn the heading pattern already in use (`## Version X.Y.Z`, `## [X.Y.Z]`, bilingual bullets or
   not, etc.) and match it exactly — never impose a different style on an existing file. If none
   exists, skip straight to step 5 (no changelog to write, but a notes fragment is still needed).
4. If a changelog file was found, write the new entry into it using the detected style, in whatever
   position new entries go (usually right after the top intro block, before the previous latest entry
   — check where that boundary falls in the existing file).
5. **Save the notes fragment for the tag** — this is what makes release notes portable across CI setups
   without CI having to re-parse any particular changelog format (see [Populating release notes from
   the tag message](#populating-release-notes-from-the-tag-message) below). Write just the new entry's
   body (no heading line, trimmed of leading/trailing blank lines) to a scratch file:
   ```bash
   NOTES_FILE="$(mktemp)"
   # write the section you just composed for $NEW_VERSION into $NOTES_FILE
   ```
   If step 3 found no changelog file, fall back to the raw commit list as the fragment instead:
   ```bash
   git log "$TAG"..HEAD --pretty=format:'- %s (%h)' > "$NOTES_FILE"
   ```
   (`$NEW_VERSION` is resolved in step 3 of the next section — compute it first if writing this in one
   pass.)

### Step 3 — Bump version fields and create the tag

1. Resolve `$NEW_VERSION`: the user-supplied argument if given, otherwise `$PREV` with its patch
   component incremented by one.
2. Update the `version` field in `manifest.json`, and in `package.json` too if that file exists and
   has one — a targeted substitution (not a JSON re-serialize, to avoid reformatting the file):
   ```bash
   sed -i '' -E "s/\"version\": \"[^\"]+\"/\"version\": \"$NEW_VERSION\"/" manifest.json
   [ -f package.json ] && sed -i '' -E "s/\"version\": \"[^\"]+\"/\"version\": \"$NEW_VERSION\"/" package.json
   ```
3. Stage `manifest.json`, `package.json` (if touched), and the changelog file (if one was written),
   then commit — match this repo's existing convention for these commits (e.g. `chore(release): bump
   version to $NEW_VERSION`, visible in its own `git log`).
4. Tag that commit as an **annotated** tag, using the notes fragment from Step 2 as the tag message —
   this is what lets CI hand back real release notes without depending on any particular changelog file
   format:
   ```bash
   git tag -a "$NEW_VERSION" -F "$NOTES_FILE"
   ```
   If `$NOTES_FILE` ended up empty for any reason, still create an annotated tag (not a bare/lightweight
   one) with a minimal fallback message, so CI logic that reads the tag's contents always finds
   something rather than silently producing an empty release body:
   ```bash
   git tag -a "$NEW_VERSION" -m "Release $NEW_VERSION"
   ```

### Step 4 — Push and hand off to CI

The version-bump commit has to reach `origin/main` — that is the commit the tag points at, and Step 0
requires the released commit to be on the default branch.

```bash
# On main:
git push origin HEAD              # pushes the bump commit to main

# On a feature/worktree branch that Step 0 confirmed is level with origin/main:
git push origin HEAD:main         # fast-forwards main to include the bump commit
git branch -f main HEAD 2>/dev/null || true   # keep local main in sync if it exists

git push origin "$NEW_VERSION"
```

If Step 0 surfaced that `HEAD` is ahead of `origin/main` and the user chose to release anyway, push
whatever they agreed to (their feature branch, or `HEAD:main`) — do not silently fast-forward `main`
past commits they have not signed off on.

Pushing the tag is what triggers the release pipeline (e.g. a GitHub Actions workflow gated on
`on: push: tags:`, see [Release 工作流检查（GitHub Actions）](#release-工作流检查github-actions) and
[Populating release notes from the tag message](#populating-release-notes-from-the-tag-message) below).
If that workflow has been wired to read the annotated tag's message, the draft release body fills in
automatically — no manual editing on GitHub needed. Report the pushed tag name and that CI has been
triggered, then stop — do not wait on or poll the CI run as part of this command.

## Report Format

```
[FILE:LINE]  <full offending line>
  Rule:  <rule name>
  Fix:   <what to change>
```

Mark items that are intentional exceptions with `[KEPT: reason]`.
