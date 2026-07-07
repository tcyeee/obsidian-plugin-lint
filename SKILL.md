---
name: obsidian-plugin-lint
description: Use when asked to lint, check, or review an Obsidian plugin's CSS or code for style violations. Triggers on "lint", "检查代码", "check styles", before committing/releasing a plugin, or when seeing submission bot warnings: "Avoid !important", "Use the full 6-digit hex format", "Unexpected duplicate", "Unexpected browser feature", "Unexpected unknown type selector", "Unsafe assignment to innerHTML", "Sets styles directly instead of using CSS classes", "Unsafe call of an error or any typed value", "This assertion is unnecessary since the receiver accepts the original type of the expression". Also handles the "/obsidian-plugin-publish [version]" release workflow — commit outstanding work, generate version.md from git history since the last release tag, bump manifest.json/package.json, tag, and push to trigger the release pipeline. Triggers on "/obsidian-plugin-publish", "发布插件", "打个 release", "bump version and tag".
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

### CSS Best Practices (official docs recommendation, not enforced by eslint-plugin)

| Practice | Violation | Fix |
|----------|-----------|-----|
| Use CSS variables | Hardcoded colors/sizes (`color: white`, `font-size: 12px`) | Use Obsidian variables: `var(--text-normal)`, `var(--font-ui-small)` |
| Avoid `!important` | Any `!important` declaration | Add `.workspace` parent selector to raise specificity |
| Full 6-digit hex | Shorthand hex `#fff`, `#aaa`, `#ccc`, `#888` | Expand to 6 digits: `#ffffff`, `#aaaaaa`, `#cccccc`, `#888888` |
| No duplicate properties | Same property declared twice in one rule block (e.g. `position: fixed` repeated) | Remove the duplicate; keep the intended declaration |
| Avoid `:has()` | Any `:has()` selector — broad selector invalidation hurts performance | Structural: target the matched element's own class. State-based: JS toggles a marker class, key CSS off it |
| Browser feature compatibility | Properties only partially supported by Obsidian's Electron/Chromium version (`clip-path`, `column-count`, `box-decoration-break`, newer `text-decoration-*` sub-properties) | Add vendor prefixes or avoid the feature; see section below |
| No unknown type selectors | Non-standard HTML element names in selectors (`webview`, `mjx-container`) | Use class selectors instead, or acknowledge as intentional third-party elements |

## Checking CSS for `!important`

**Submission bot warning:** `Avoid !important — override styles by increasing selector specificity or using CSS variables instead.`

```bash
grep -n '!important' styles.css
```

For each hit:

1. Try adding `.workspace` as parent — outranks Obsidian's base rules (which don't use `!important` on these properties)
2. If Obsidian itself uses `!important` on the same property, keep it and document why

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

Two patterns, two different fixes:

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

## TypeScript 类型安全

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

### 废弃 API → eslint-disable 或迁移

Obsidian 1.13.0 将 `PluginSettingTab.display()` 标为 `@deprecated`，推荐改用 `getSettingDefinitions()`。迁移需要完整重写 Settings Tab（声明式 API）。在迁移前，用带说明的 `eslint-disable-next-line` 压制警告：

```typescript
// ❌
this.display();

// ✅ — suppress until the tab is migrated to getSettingDefinitions()
// eslint-disable-next-line @typescript-eslint/no-deprecated -- PluginSettingTab.display() deprecated in 1.13.0; migrate to getSettingDefinitions() when refactoring settings
this.display();
```

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
| `minAppVersion` | 低于代码中实际使用的 API 所需版本 | 升高 `minAppVersion` 至 API 要求的最低版本 |

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

```json
// ❌
"description": "Image Cluster helps you combine multiple images in your notes."

// ✅
"description": "Combine multiple images together in your notes for a more organized layout."
```

检查方法：`cat manifest.json` 查看 `description` 字段是否以插件 `name` 开头。

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

### Populating release notes from the tag message

不同项目的发布机制不统一——有的直接调 `gh release create`，有的用第三方 Action（如
`softprops/action-gh-release`）。为了让"draft release 自动带上本次更新内容"这件事能跨项目复用，
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

```yaml
- name: Create release
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    tag="${GITHUB_REF#refs/tags/}"
    notes="$(git tag -l --format='%(contents)' "$tag")"

    if [ -n "$notes" ]; then
      echo "$notes" > /tmp/release-notes.md
      gh release create "$tag" --title="$tag" --draft --notes-file /tmp/release-notes.md <assets...>
    else
      gh release create "$tag" --title="$tag" --draft --generate-notes <assets...>
    fi
```

（`<assets...>` 替换为该项目实际发布的文件列表，例如 `main.js manifest.json styles.css`。）

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

```bash
# Find affected properties
grep -rn 'clip-path\|mask-image\|mask:\|column-count\|column-width\|box-decoration-break\|text-indent.*each-line\|text-indent.*hanging\|text-decoration-thickness\|text-underline-offset\|text-decoration-skip-ink\|text-underline-position' styles.css theme/
```

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
| Recommendation | Vault Enumeration | `vault.getFiles()`、`vault.getMarkdownFiles()`、`vault.getAllLoadedFiles()` | 通常为必要功能（链接解析、搜索）；确认范围最小化，无需移除 |
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

```bash
git push origin HEAD    # the version-bump commit, so main doesn't diverge from what's tagged
git push origin "$NEW_VERSION"
```

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
