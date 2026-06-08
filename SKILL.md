---
name: obsidian-plugin-lint
description: Use when asked to lint, check, or review an Obsidian plugin's CSS or code for style violations. Triggers on "lint", "检查代码", "check styles", before committing/releasing a plugin, or when seeing submission bot warnings: "Avoid !important", "Use the full 6-digit hex format", "Unexpected duplicate".
---

# Obsidian Plugin Lint

## Overview

Static analysis pass for Obsidian plugin code. Covers both official Obsidian ESLint rules and general CSS best practices. Run before every release.

## Rules

### Official Obsidian ESLint Plugin Rules (`obsidianmd/eslint-plugin`)

Source: https://github.com/obsidianmd/eslint-plugin

| Rule | What to check | Fix |
|------|---------------|-----|
| `no-static-styles-assignment` | `el.style.color = ...`, `el.style.cssText = ...` (any inline style via JS) | CSS class: `el.addClass('my-class')`. For dynamic values: `setCssProps(el, { '--my-var': value })` |
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

Any hit that assigns a value (`.style.color =`, `.style.display =`, `.style.cssText =`) is a violation. Use a CSS class for static values; for dynamic CSS properties use `setCssProps`:

```typescript
// ❌
el.style.cssText = 'position: absolute; top: 0';
el.style.color = 'red';

// ✅ static
el.addClass('my-positioned-el');

// ✅ dynamic
setCssProps(el, { '--offset-top': `${y}px` });
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

### `loadData()` 返回 `any` 导致不安全赋值

`Plugin.loadData()` 返回 `Promise<any>`，直接赋值给强类型字段会触发 `@typescript-eslint/no-unsafe-assignment`：

```typescript
// ❌ — Unsafe assignment of an `any` value
this.settings = Object.assign({}, DEFAULT_SETTINGS, await this.loadData());

// ✅ 加类型断言
this.settings = Object.assign({}, DEFAULT_SETTINGS, (await this.loadData()) as Partial<MyPluginSettings>);
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
const re = /[ -]/;
```

**多余转义**：`\S` 在字符类 `[...]` 外部是合法的，但某些位置会触发 `no-useless-escape`。去掉多余的反斜杠：

```typescript
// ❌  \. inside a character class is just a literal dot — \. → .
const re = /[\.\s]/;

// ✅
const re = /[.\s]/;
```

```bash
grep -n 'eslint.*no-useless-escape\|\\\\[^nrtuvxuU0-9$()*+.?[\]^{|}\\]' src/*.ts
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

## Report Format

```
[FILE:LINE]  <full offending line>
  Rule:  <rule name>
  Fix:   <what to change>
```

Mark items that are intentional exceptions with `[KEPT: reason]`.
