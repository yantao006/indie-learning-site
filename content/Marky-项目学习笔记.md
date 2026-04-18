
> 基于对 [Marky](https://github.com/GRVYDEV/marky) 项目的完整代码库分析整理。
> 涵盖 Tauri 桌面框架原理、Markdown 渲染管线、Rust/React 协同架构、以及工程设计决策。

---

## 目录

1. [项目全景](#1-项目全景)
2. [Tauri 框架原理：不需要 Xcode 也能做 macOS App](#2-tauri-框架原理不需要-xcode-也能做-macos-app)
3. [整体技术栈](#3-整体技术栈)
4. [代码结构](#4-代码结构)
5. [Markdown 可视化渲染管线（核心）](#5-markdown-可视化渲染管线核心)
6. [语法高亮：Shiki 的使用方式](#6-语法高亮shiki-的使用方式)
7. [Mermaid 图表渲染](#7-mermaid-图表渲染)
8. [文档内搜索的 DOM 操作技巧](#8-文档内搜索的-dom-操作技巧)
9. [前端架构：组件设计与状态管理](#9-前端架构组件设计与状态管理)
10. [Rust 后端设计](#10-rust-后端设计)
11. [前后端 IPC 通信](#11-前后端-ipc-通信)
12. [主题系统](#12-主题系统)
13. [样式架构：两层分离](#13-样式架构两层分离)
14. [安全设计](#14-安全设计)
15. [工程约定与边界](#15-工程约定与边界)
16. [核心设计决策与权衡](#16-核心设计决策与权衡)
17. [可复用的技巧总结](#17-可复用的技巧总结)

---

## 1. 项目全景

### 是什么

Marky 是一个 **Tauri v2 桌面 Markdown 查看器**，支持：

- 命令行 `marky FILE.md` 或 `marky FOLDER/` 启动
- 文件夹持久化管理（Obsidian 风格左侧边栏）
- 多标签页 + 分栏布局
- Markdown 全功能渲染：表格、代码块、任务列表、数学公式、Mermaid 图表
- `Cmd+K` 命令面板 + 模糊文件搜索
- `Cmd+F` 文档内搜索
- 亮色 / 暗色 / 跟随系统主题

### 定位

**只读查看器**，不做编辑功能。主要用途是查看 Claude 生成的计划文档等 Markdown 文件。

### 核心技术栈

| 层 | 技术 |
|---|---|
| 桌面框架 | Tauri v2（Rust + 系统 WebView） |
| 前端 | React 19 + TypeScript 5.8 + Vite 7 |
| Markdown | markdown-it + Shiki + Mermaid + DOMPurify |
| UI 组件 | shadcn/ui + Radix UI + Tailwind CSS 4 |
| 后端 | Rust 2021：nucleo / notify / serde |
| 打包 | Tauri CLI → `.app` + `.dmg` |

---

## 2. Tauri 框架原理：不需要 Xcode 也能做 macOS App

### 核心思路

Tauri 把"原生桌面 App"分成两层：

```
┌─────────────────────────────────────────────────────┐
│              Web 前端（你写的 React/Vue/任意框架）     │
│         运行在系统自带的 WebView 里（WKWebView）        │
├─────────────────────────────────────────────────────┤
│              Rust 主进程（你写的业务逻辑）              │
│         文件读写 / CLI 解析 / 搜索 / 设置持久化          │
├─────────────────────────────────────────────────────┤
│              Tauri Rust Core（框架层）                │
│    窗口管理 / IPC / 插件 / 打包 → macOS NSWindow       │
└─────────────────────────────────────────────────────┘
```

关键：**UI 渲染完全交给操作系统自带的 WebView**，不像 Electron 需要内嵌完整 Chromium。

### Electron vs Tauri 对比

| | Electron | Tauri |
|---|---|---|
| UI 渲染 | 内嵌 Chromium（~150MB）| 系统 WebView（0 额外体积）|
| 后端语言 | Node.js | Rust |
| 安装包大小 | ~80–150MB | ~3–15MB |
| 内存占用 | ~100–300MB | ~30–80MB |
| 跨平台 | ✅ | ✅ |
| 典型产品 | VS Code、Slack、Discord | 1Password、Zed 编辑器 |

### 为什么不需要 Xcode IDE

1. **编译**：Rust 代码用 `cargo build` 编译成标准 macOS 二进制，不需要 Xcode 项目
2. **打包**：`tauri-build` crate 在 `build.rs` 里自动生成 `Info.plist`、组织 `.app` 目录结构
3. **图标**：`tauri.conf.json` 里指定 `.icns` 文件路径，CLI 自动处理
4. **只需要**：Xcode Command Line Tools（`xcode-select --install`，约 200MB，提供 `clang` + macOS SDK headers）

### `.app` 的真实结构

macOS 的 `.app` bundle 就是一个普通目录：

```
Marky.app/
└── Contents/
    ├── Info.plist          ← 元数据（自动生成）
    ├── MacOS/
    │   └── marky           ← Rust 编译出的二进制
    └── Resources/
        ├── _up_/           ← Vite 打包的 React 资源
        │   ├── index.html
        │   └── assets/
        └── icon.icns
```

### `tauri.conf.json` 替代整个 Xcode 项目

```json
{
  "productName": "Marky",
  "identifier": "dev.marky.app",
  "build": {
    "beforeDevCommand": "pnpm dev",
    "beforeBuildCommand": "pnpm build",
    "frontendDist": "../dist"
  },
  "bundle": {
    "targets": "all",
    "icon": ["icons/icon.icns"]
  }
}
```

一条命令完成全流程：

```bash
pnpm tauri build
# ① pnpm build      → Vite 编译 React 到 dist/
# ② cargo build     → Rust 编译二进制
# ③ tauri-cli 打包  → 生成 Marky.app + Marky.dmg
```

### CLI 命令实现原理

安装脚本 `scripts/install-cli.sh` 在 `~/.local/bin/marky` 写一个 shell wrapper：

```bash
# 有 .app bundle 时（推荐）：
exec /usr/bin/open -a "/path/to/Marky.app" --args "$@"

# 只有原始二进制时：
nohup "/path/to/marky" "$@" >/dev/null 2>&1 &
disown
```

`open -a` 是 macOS 系统命令，通过 Launch Services 启动 App，传入额外参数。
Rust 的 `cli.rs` 读取 `std::env::args()` 获取文件路径，然后通知前端打开。

---

## 3. 整体技术栈

### 前端

| 库 | 版本 | 职责 |
|---|---|---|
| React | 19 | UI 组件，函数组件 + hooks |
| TypeScript | 5.8 | 类型系统，严格模式 |
| Vite | 7 | 打包 + HMR，端口固定 1420 |
| markdown-it | 14 | Markdown 解析，单例 |
| Shiki | 1 | 语法高亮，懒加载单例 |
| Mermaid | 11 | 流程图 / 时序图，懒加载 |
| DOMPurify | 3 | HTML 消毒，防 XSS |
| shadcn/ui | — | 组件库，复制进项目 |
| Radix UI | — | Headless 无障碍原语 |
| cmdk | 1 | Command palette 引擎 |
| Tailwind CSS | 4 | 工具类样式 |
| lucide-react | — | 图标 |

### Rust 后端

| 库 | 职责 |
|---|---|
| tauri | 窗口 / IPC / 命令注册 |
| nucleo | 模糊搜索引擎（Helix 同款）|
| notify + debouncer | 文件系统监听，debounce 200ms |
| serde / serde_json | JSON 序列化，IPC 边界 |
| anyhow / thiserror | 错误处理 |
| uuid | folder ID 生成 |
| parking_lot | 高性能 Mutex |
| dirs | 获取系统 app_data_dir |
| ignore | 文件遍历时应用 .gitignore 规则 |

### Tauri 插件

```toml
tauri-plugin-opener        # 打开系统浏览器/文件管理器
tauri-plugin-dialog        # 系统文件选择对话框
tauri-plugin-single-instance  # 防止多实例，第二实例重定向到当前窗口
```

---

## 4. 代码结构

### 目录布局

```
marky/
├── src/                    前端 React
│   ├── App.tsx             根组件，全局状态 + 布局
│   ├── main.tsx            React DOM 挂载
│   ├── components/         业务组件
│   │   ├── Viewer.tsx      核心渲染组件
│   │   ├── Pane.tsx        内容区（TabBar + Viewer + DocSearch）
│   │   ├── FolderSidebar.tsx  左侧文件夹面板
│   │   ├── FileTree.tsx    递归文件树
│   │   ├── TabBar.tsx      标签条
│   │   ├── Toolbar.tsx     顶部工具栏
│   │   ├── TableOfContents.tsx  右侧目录
│   │   ├── DocSearch.tsx   文档内搜索（Cmd+F）
│   │   ├── CommandPalette.tsx  命令面板（Cmd+K）
│   │   ├── CodeCopyOverlay.tsx  代码块复制按钮
│   │   └── ui/             shadcn 组件原语
│   ├── lib/                工具库
│   │   ├── markdown.ts     markdown-it 单例 + 插件
│   │   ├── highlight.ts    Shiki 懒加载单例
│   │   ├── mermaid.ts      Mermaid 懒加载
│   │   ├── workspace.ts    tabs/panes 纯 reducer
│   │   ├── tauri.ts        invoke() 类型化包装
│   │   ├── theme.tsx       ThemeProvider + useTheme()
│   │   ├── docSearch.ts    文档搜索 DOM 操作
│   │   └── utils.ts        cn() 等工具函数
│   ├── styles/
│   │   ├── index.css       全局 + shadcn CSS 变量
│   │   └── markdown.css    Markdown prose 全部样式
│   └── types/
│       └── shims.d.ts      类型补丁
│
├── src-tauri/              Rust 后端
│   ├── tauri.conf.json     打包配置
│   ├── Cargo.toml          Rust 依赖 + release 优化
│   ├── build.rs            tauri_build::build()
│   ├── capabilities/       Tauri 权限声明
│   └── src/
│       ├── main.rs         2 行入口
│       ├── lib.rs          Tauri Builder + 插件 + invoke_handler
│       ├── cli.rs          CLI 参数解析
│       ├── commands.rs     所有 #[tauri::command]
│       ├── registry.rs     Arc<RwLock<FolderRegistry>>
│       ├── folder.rs       文件树遍历 + 忽略规则
│       ├── search.rs       nucleo 模糊搜索
│       ├── watcher.rs      notify 文件监听
│       ├── settings.rs     settings.json 读写
│       ├── fs.rs           read_file UTF-8
│       └── error.rs        AppError + thiserror
│
├── scripts/
│   └── install-cli.sh      安装 CLI wrapper 到 ~/.local/bin
└── vite.config.ts          Vite + Vitest 配置
```

### 组件职责速查

| 组件 | 职责 |
|---|---|
| `App.tsx → AppShell` | 全局状态（useReducer）/ 键盘快捷键 / 文件拖放 / folder 同步 |
| `FolderSidebar` | folder 列表 + 文件树 CRUD |
| `FileTree` | 递归渲染 TreeNode，点击触发 `openFile()` |
| `Toolbar` | 面包屑 / 打开文件 / 分栏按钮 / 触发 Cmd+F |
| `Pane` | TabBar + Viewer + DocSearch overlay，维护 `contentNonce` |
| `TabBar` | 水平标签条，close / select |
| `Viewer` | **核心**：3 个 Effect 编排整个渲染管线 |
| `TableOfContents` | `extractHeadings()` 提取 H1-H4，点击跳锚点 |
| `DocSearch` | Cmd+F 文档内搜索，高亮 `<mark>` |
| `CommandPalette` | Cmd+K：文件搜索 + 命令 + 主题切换 |
| `CodeCopyOverlay` | 给每个 `<pre>` 注入 Copy 按钮（纯 DOM 变更）|

---

## 5. Markdown 可视化渲染管线（核心）

Marky 的渲染是整个项目最复杂的部分，分三个阶段顺序执行。

### 整体流程

```
Raw Markdown string
       ↓  (Effect 1 — 同步)
markdown-it.render()
       ↓
DOMPurify.sanitize()
       ↓
dangerouslySetInnerHTML   ← 初始 HTML 注入 DOM，代码块仍是纯文本
       ↓  (Effect 2 — 异步，可取消)
   ┌───┴───────┬───────────┐
Shiki 高亮   Mermaid     Copy 按钮
pre 替换     pending→SVG  注入到 pre
       ↓  (Effect 3)
滚动位置恢复
```

### `src/lib/markdown.ts`：配置 markdown-it 单例

```typescript
const md = new MarkdownIt({
  html: true,       // 允许 markdown 里内嵌 HTML
  linkify: true,    // 自动识别 URL
  typographer: true // 智能引号等排版优化
});

// 插件 1：给标题加锚点 ID + 永久链接
md.use(anchor, {
  permalink: anchor.permalink.linkInsideHeader({ symbol: "#" }),
  slugify: (s) => s.toLowerCase().trim()
    .replace(/[^a-z0-9\s-]/g, "")
    .replace(/\s+/g, "-"),
});

// 插件 2：脚注 [^1]
md.use(footnote);

// 插件 3：任务列表 - [x]
md.use(taskLists, { enabled: true, label: false });

// 规则覆盖 1：Mermaid 块不走高亮，输出特殊 class 留待后处理
const defaultFence = md.renderer.rules.fence!;
md.renderer.rules.fence = (tokens, idx, options, env, self) => {
  if (tokens[idx].info.trim() === "mermaid") {
    const escaped = token.content.replace(/&/g, "&amp;")...;
    return `<pre class="mermaid-pending"><code>${escaped}</code></pre>`;
  }
  return defaultFence(tokens, idx, options, env, self);
};

// 规则覆盖 2：外部链接加 target=_blank
md.renderer.rules.link_open = (tokens, idx, options, env, self) => {
  const href = tokens[idx].attrGet("href") || "";
  if (/^https?:\/\//i.test(href)) {
    tokens[idx].attrSet("target", "_blank");
    tokens[idx].attrSet("rel", "noreferrer noopener");
  }
  return defaultLink(tokens, idx, options, env, self);
};
```

**关键约定**：全项目只有这一个 markdown-it 实例，所有组件通过 `renderMarkdown()` 和 `extractHeadings()` 调用。

### `src/components/Viewer.tsx`：三个 Effect 的编排

**Effect 1 — 解析（同步）**

```typescript
React.useEffect(() => {
  setHtml(renderMarkdown(source));  // markdown-it → DOMPurify → setState
}, [source]);
```

**Effect 2 — 富化（异步，有取消机制）**

```typescript
React.useEffect(() => {
  const root = ref.current;
  let cancelled = false;

  (async () => {
    // Step 1: Shiki 代码高亮
    const codeBlocks = root.querySelectorAll("pre > code[class*='language-']");
    for (const code of Array.from(codeBlocks)) {
      const lang = code.className.match(/language-([\w+-]+)/)?.[1];
      const highlighted = await highlightCode(code.textContent, lang, resolved);
      if (cancelled) return;  // 若组件已卸载，立即退出
      // 用 <template> 原地替换 <pre>
      const tpl = document.createElement("template");
      tpl.innerHTML = highlighted.trim();
      code.parentElement?.replaceWith(tpl.content.firstElementChild);
    }

    if (cancelled) return;

    // Step 2: 注入 Copy 按钮
    attachCopyButtons(root);

    // Step 3: Mermaid 渲染
    await renderMermaidBlocks(root, resolved);

    // 通知父组件渲染完成（更新 contentNonce，触发 DocSearch 重扫）
    if (!cancelled) onRenderedRef.current?.();
  })();

  return () => { cancelled = true; };
}, [html, resolved]);  // html 或主题变化时重新执行
```

**Effect 3 — 滚动位置恢复**

```typescript
// 模块级 Map，跨渲染周期持久
const scrollMemory = new Map<string, number>();

React.useEffect(() => {
  const el = scrollerRef.current;
  const saved = scrollMemory.get(filePath) ?? 0;
  el.scrollTop = saved;
  const onScroll = () => scrollMemory.set(filePath, el.scrollTop);
  el.addEventListener("scroll", onScroll, { passive: true });
  return () => el.removeEventListener("scroll", onScroll);
}, [filePath, html]);
```

### 关键实现细节

**为什么 `dangerousHtml` 要 `useMemo`？**

```typescript
// 正确：引用稳定，html 不变时 React 不会重新 set innerHTML
const dangerousHtml = React.useMemo(() => ({ __html: html }), [html]);

// 错误：每次渲染都产生新对象，React 会重新执行 innerHTML，
// 抹掉 Shiki 刚注入的 token spans，导致代码块闪回纯文本
const dangerousHtml = { __html: html };  // ❌
```

**为什么 `onRendered` 要用 ref？**

```typescript
// 父组件每次 re-render 都会传入新的函数引用
// 如果 onRendered 进入 Effect deps，会导致高亮 Effect 不停重跑
const onRenderedRef = React.useRef(onRendered);
React.useEffect(() => {
  onRenderedRef.current = onRendered;  // 始终保持最新值
}, [onRendered]);
// Effect 里通过 ref 读取，不把 onRendered 加入 deps
```

**`<template>` 元素替换技巧**

```typescript
// Shiki 返回完整 HTML 字符串，需要转换成 DOM 节点
const tpl = document.createElement("template");
tpl.innerHTML = highlighted.trim();
const replacement = tpl.content.firstElementChild;
if (replacement) pre.replaceWith(replacement);

// 为什么用 template 而不是 innerHTML？
// template.content 是 DocumentFragment，解析时不执行脚本、不加载资源，安全且高效
```

---

## 6. 语法高亮：Shiki 的使用方式

### `src/lib/highlight.ts`：懒加载单例

```typescript
const COMMON_LANGS = ["ts", "tsx", "js", "jsx", "json", "rust", "python",
  "go", "bash", "yaml", "toml", "html", "css", "sql", "md", "diff",
  "java", "c", "cpp", "ruby"];

const THEMES = ["github-light", "github-dark"];

let promise: Promise<Highlighter> | null = null;

// 懒加载：第一次调用时创建，后续复用同一个 Promise
export function getHighlighter(): Promise<Highlighter> {
  if (!promise) {
    promise = createHighlighter({ themes: THEMES, langs: COMMON_LANGS });
  }
  return promise;
}
```

### 按需加载不常见语言

```typescript
export async function highlightCode(code, lang, theme) {
  const hl = await getHighlighter();
  const loaded = hl.getLoadedLanguages();

  let resolvedLang = lang && loaded.includes(lang) ? lang : "";

  // 如果语言未预加载，尝试动态加载
  if (lang && !resolvedLang) {
    try {
      await hl.loadLanguage(lang);
      resolvedLang = lang;
    } catch {
      resolvedLang = "";  // 不认识的语言，降级为纯文本
    }
  }

  return hl.codeToHtml(code, {
    lang: resolvedLang || "text",
    theme: theme === "dark" ? "github-dark" : "github-light",
  });
}
```

**技巧**：Shiki 把主题颜色直接写成每个 `<span>` 的 `style` 属性，不需要额外的 CSS 映射，CSS 只需要保持 `pre.shiki` 的 `background: transparent`。

---

## 7. Mermaid 图表渲染

### 两段式处理的必要性

直接在 markdown-it 解析时渲染 Mermaid 不可行：
- markdown-it 是同步的，`mermaid.render()` 是异步的
- Mermaid 库很大（~2MB），应该懒加载

解决方案：解析时**打标记**，DOM ready 后**异步渲染**。

### `src/lib/mermaid.ts`

```typescript
let initialized = false;
let mermaidPromise: Promise<typeof import("mermaid").default> | null = null;

async function loadMermaid() {
  if (!mermaidPromise) {
    // 动态 import，只在第一次有 mermaid 块时才加载这 2MB
    mermaidPromise = import("mermaid").then((m) => m.default);
  }
  return mermaidPromise;
}

export async function renderMermaidBlocks(root: HTMLElement, theme: "light" | "dark") {
  const blocks = root.querySelectorAll<HTMLPreElement>("pre.mermaid-pending");
  if (blocks.length === 0) return;  // 没有 mermaid 块就不加载库

  const mermaid = await loadMermaid();
  mermaid.initialize({
    startOnLoad: false,
    theme: theme === "dark" ? "dark" : "default",
  });

  let i = 0;
  for (const pre of Array.from(blocks)) {
    const source = pre.textContent || "";
    const id = `mermaid-${Date.now()}-${i++}`;  // 唯一 ID
    const wrapper = document.createElement("div");
    wrapper.className = "mermaid-block";

    try {
      const { svg } = await mermaid.render(id, source);
      wrapper.innerHTML = svg;
    } catch (err) {
      wrapper.textContent = `Mermaid render error: ${err.message}`;
    }

    pre.replaceWith(wrapper);
  }
}
```

---

## 8. 文档内搜索的 DOM 操作技巧

`Cmd+F` 的搜索高亮不用 React 节点，直接操作 DOM，效率更高。

### `src/lib/docSearch.ts` 核心算法

```typescript
export function highlightMatches(root: HTMLElement, query: string): SearchHandle {
  const lowered = query.toLowerCase();

  // 收集所有文本节点（排除 SCRIPT/STYLE 和已高亮的节点）
  const targets: Text[] = [];
  const collect = (node: Node) => {
    if (node.nodeType === Node.TEXT_NODE) {
      const parent = node.parentElement;
      if (parent?.tagName === "SCRIPT" || parent?.tagName === "STYLE") return;
      if (parent?.classList?.contains("doc-search-match")) return;  // 避免重复处理
      targets.push(node as Text);
      return;
    }
    for (const child of Array.from(node.childNodes)) collect(child);
  };
  collect(root);

  const created: HTMLElement[] = [];

  for (const text of targets) {
    const value = text.nodeValue ?? "";
    const lower = value.toLowerCase();
    let cursor = 0;
    let idx = lower.indexOf(lowered, cursor);
    if (idx === -1) continue;

    // 用 DocumentFragment 批量构建替换内容
    const frag = document.createDocumentFragment();
    while (idx !== -1) {
      if (idx > cursor) frag.appendChild(document.createTextNode(value.slice(cursor, idx)));

      const mark = document.createElement("mark");
      mark.className = "doc-search-match";
      mark.textContent = value.slice(idx, idx + query.length);
      frag.appendChild(mark);
      created.push(mark);

      cursor = idx + query.length;
      idx = lower.indexOf(lowered, cursor);
    }
    if (cursor < value.length) frag.appendChild(document.createTextNode(value.slice(cursor)));

    text.parentNode?.replaceChild(frag, text);
  }

  return {
    matches: created,
    // 清理：把每个 <mark> 替换回原始文本节点
    clear: () => {
      for (const el of created) {
        const parent = el.parentNode;
        if (!parent) continue;
        parent.replaceChild(document.createTextNode(el.textContent ?? ""), el);
        parent.normalize?.();  // 合并相邻文本节点
      }
    },
  };
}
```

### `contentNonce` 模式

问题：Shiki 高亮完成后，`Viewer` 内部 DOM 变了，`DocSearch` 需要重新扫描以找到高亮后的文本节点。

解决方案：父组件 `Pane.tsx` 维护一个 `contentNonce` 计数器，Viewer 渲染完成时触发 `onRendered` 回调递增它，DocSearch 监听这个 nonce 重新执行高亮逻辑。

```typescript
// Pane.tsx
const [contentNonce, setContentNonce] = useState(0);

<Viewer onRendered={() => setContentNonce((n) => n + 1)} />
<DocSearch contentNonce={contentNonce} />

// DocSearch.tsx
useEffect(() => {
  // contentNonce 变化 → 重新扫描 DOM 高亮匹配
  const h = highlightMatches(containerRef.current, query);
}, [query, open, contentNonce, containerRef]);
```

---

## 9. 前端架构：组件设计与状态管理

### `src/lib/workspace.ts`：纯 Reducer 状态管理

不用 zustand/redux，用内置 `useReducer`，状态结构简单清晰：

```typescript
interface WorkspaceState {
  tabs: Record<string, TabState>;   // 所有打开的文件
  panes: PaneState[];               // 1 或 2 个分栏
  activePaneId: string;
  split: "horizontal" | "vertical" | null;
  nextTabId: number;
  nextPaneId: number;
}
```

纯函数 reducer，易于测试：

```typescript
export function reduce(state: WorkspaceState, action: Action): WorkspaceState {
  switch (action.type) {
    case "OPEN_FILE": {
      // 去重：如果文件已打开，直接切换到那个 tab
      const existing = findTabByPath(state, action.path);
      if (existing) return { ...withPane(state, existing.paneId, ...), ... };
      // 否则新建 tab
      ...
    }
    case "SPLIT": {
      // 新 pane 故意留空，不复制当前 tab
      // 因为两个 Viewer 同时高亮同一 source 会互相 race，锁住渲染
      return { ...state, panes: [...state.panes, { id: newPaneId, tabIds: [], activeTabId: null }], ... };
    }
  }
}
```

**Compact 函数**：每次 action 后清理空 pane，保持 "最多 2 个 pane，没有内容的 pane 自动消失" 的不变式。

### 键盘快捷键集中管理

所有全局快捷键在 `App.tsx` 一处注册：

```typescript
useEffect(() => {
  const onKey = (e: KeyboardEvent) => {
    const meta = e.metaKey || e.ctrlKey;
    if (meta && e.key === "k") { e.preventDefault(); setPaletteOpen(v => !v); }
    if (meta && e.key === "o") { e.preventDefault(); handlePickFile(); }
    if (meta && e.key === "f") { e.preventDefault(); setSearchPaneId(state.activePaneId); }
    if (meta && e.key === "\\") {
      e.preventDefault();
      dispatch({ type: "SPLIT", direction: e.shiftKey ? "horizontal" : "vertical" });
    }
  };
  window.addEventListener("keydown", onKey);
  return () => window.removeEventListener("keydown", onKey);
}, [state.activePaneId, activePane.activeTabId, activePane.id]);
```

### 文件拖放

```typescript
useEffect(() => {
  let unlisten: (() => void) | undefined;
  (async () => {
    // 使用 Tauri 的 drag-drop 事件，不是 HTML5 拖放 API
    const off = await getCurrentWebview().onDragDropEvent((e) => {
      if (e.payload.type === "drop" && e.payload.paths.length > 0) {
        openFile(e.payload.paths[0]);  // 取第一个拖入文件
      }
    });
    unlisten = off;
  })();
  return () => unlisten?.();
}, [openFile]);
```

---

## 10. Rust 后端设计

### 保持 `main.rs` 极简

```rust
// main.rs — 只有 2 行
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

fn main() {
    marky_lib::run()
}
```

所有逻辑在 `lib.rs` 的 `pub fn run()` 里，这样可以被单元测试引用，也支持移动端入口点（`#[cfg_attr(mobile, tauri::mobile_entry_point)]`）。

### 状态集中化：`Arc<RwLock<FolderRegistry>>`

```rust
// lib.rs — 所有 folder 状态的唯一来源
let registry: SharedRegistry = Arc::new(FolderRegistry::from_settings(settings));
let watchers: SharedWatchers = Arc::new(Watchers::default());

app.manage(registry);
app.manage(watchers);
```

不散落在各模块，任何 `#[tauri::command]` 通过 `State<SharedRegistry>` 参数获取。

### 命令注册

```rust
.invoke_handler(tauri::generate_handler![
    commands::get_initial_target,
    commands::read_file,
    commands::list_folders,
    commands::add_folder,
    commands::search_files,
    // ...
])
```

`tauri::generate_handler!` 宏生成类型安全的分发代码，JSON 序列化自动处理。

### Rust Release 优化配置

```toml
[profile.release]
panic = "abort"       # panic 直接终止，不展开栈（更小体积）
codegen-units = 1     # 单编译单元，更好的跨函数优化
lto = true            # 链接时优化，进一步消除死代码
opt-level = "s"       # 优化体积而非速度（桌面工具更注重体积）
strip = true          # 去掉调试符号
```

### 文件夹忽略规则

```rust
// folder.rs
const DEFAULT_IGNORES: &[&str] = &[
    ".git", "node_modules", "target", "dist", "build",
    ".DS_Store", "*.lock"
];
```

使用 `ignore` crate（`ripgrep` 同款）支持 `.gitignore` 语义，比手写过滤逻辑健壮得多。

---

## 11. 前后端 IPC 通信

### 类型化的 `invoke()` 包装（`src/lib/tauri.ts`）

```typescript
// 所有 invoke() 调用集中在这一个文件
// 组件永远不直接调用 @tauri-apps/api 的 invoke

export const tauri = {
  getInitialTarget: () => invoke<InitialTarget>("get_initial_target"),
  readFile: (path: string) => invoke<string>("read_file", { path }),
  listFolders: () => invoke<Folder[]>("list_folders"),
  addFolder: (path: string) => invoke<Folder>("add_folder", { path }),
  searchFiles: (query: string, limit = 50) =>
    invoke<SearchResult[]>("search_files", { args: { query, limit } }),
  saveTheme: (theme: string) => invoke<void>("save_theme", { theme }),
};
```

这样做的好处：
1. 接口变更只改一处
2. TypeScript 类型集中定义
3. 组件代码不暴露实现细节

### 事件监听

```typescript
// 事件也集中在 tauri.ts 里封装
export function onFolderChanged(cb: (folderId: string) => void): Promise<UnlistenFn> {
  return listen<string>("folder://changed", (e) => cb(e.payload));
}

// 使用时注意清理
useEffect(() => {
  const off = onFolderChanged(() => refreshFolders());
  return () => { off.then(fn => fn()); };  // 组件卸载时取消监听
}, []);
```

### Rust 端推送事件

```rust
// watcher.rs — 文件变更时推送给前端
tauri::Emitter::emit(app_handle, "folder://changed", &folder_id)?;

// lib.rs — 第二实例启动时推送目标
tauri::Emitter::emit(app, "cli://target", &resolved)?;
```

### 绝对路径约定

> 前后端 IPC 边界传递的文件路径**始终是绝对路径字符串**，永远不用相对路径。

这消除了工作目录不一致的歧义（`open -a` 启动的 App 工作目录是 `/`）。

---

## 12. 主题系统

### `src/lib/theme.tsx`：React Context + localStorage

```typescript
function applyTheme(theme: Theme): "light" | "dark" {
  const resolved = theme === "system"
    ? (window.matchMedia("(prefers-color-scheme: dark)").matches ? "dark" : "light")
    : theme;
  // shadcn 的惯例：切换 html 元素的 dark class
  document.documentElement.classList.toggle("dark", resolved === "dark");
  return resolved;
}
```

**`resolved` 的意义**：组件只关心最终是 light 还是 dark（`useTheme().resolved`），不关心 "system" 这个中间状态。Shiki 和 Mermaid 都接受 `resolved` 作为主题参数。

### 主题联动

```typescript
// Viewer.tsx — Effect 2 的依赖包含 resolved
React.useEffect(() => {
  // ...高亮代码...
  await highlightCode(text, lang, resolved);       // ← 主题变了要重新高亮
  await renderMermaidBlocks(root, resolved);       // ← 主题变了要重新渲染
}, [html, resolved]);  // resolved 变化触发完整重渲染
```

---

## 13. 样式架构：两层分离

### 分离原则

```
Tailwind CSS 工具类  →  组件 chrome（布局、边距、交互状态）
markdown.css         →  .markdown-body 下所有 HTML 元素的排版样式
```

原因：Tailwind 类和 prose 样式混在一起很难维护。markdown-it 生成的 HTML 不带 CSS 类（只有标准 HTML 标签），只能用后代选择器来样式化。

### `src/styles/markdown.css` 结构

```css
.markdown-body {
  font-size: 15px;
  max-width: 860px;
  margin: 0 auto;
  padding: 2rem 2.5rem 6rem;
}

/* 标题 */
.markdown-body h1 {
  font-size: 2rem;
  border-bottom: 1px solid var(--border);  /* ← 使用 shadcn CSS 变量 */
}

/* 代码块 */
.markdown-body pre.shiki { padding: 1rem 1.1rem; }
.markdown-body pre.shiki code { background: transparent; padding: 0; }

/* Mermaid */
.markdown-body .mermaid-block {
  background: var(--muted);
  border: 1px solid var(--border);
  border-radius: 0.5rem;
  text-align: center;
}

/* 搜索高亮 */
.markdown-body mark.doc-search-match {
  background: color-mix(in oklch, var(--muted-foreground) 35%, transparent);
}
.markdown-body mark.doc-search-active {
  background: var(--primary);
  color: var(--primary-foreground);
}
```

### shadcn CSS 变量 + Tailwind 的配合

shadcn 定义的 CSS 变量（`--background`、`--foreground`、`--primary` 等）在 `src/styles/index.css` 里声明，`markdown.css` 直接引用这些变量——这样亮暗主题切换时，prose 样式自动跟着变。

---

## 14. 安全设计

### DOMPurify 的使用

```typescript
// markdown.ts — 每次渲染都过一遍消毒
export function renderMarkdown(source: string): string {
  const html = md.render(source);
  return DOMPurify.sanitize(html, {
    ADD_ATTR: ["target", "class", "id", "aria-hidden"],  // 这些属性需要保留
    ADD_TAGS: ["section"],
  });
}
```

Markdown 文件可能包含任意 HTML，用户打开不信任的文件时需要防 XSS。

### Tauri Capabilities 权限模型

```json
// capabilities/default.json
{
  "permissions": [
    "core:default",
    "opener:default",     // 打开系统浏览器
    "dialog:default",     // 文件选择对话框
    "core:event:allow-listen"
    // 注意：没有 fs:write，Marky 是只读查看器
    // WebView 里的 JS 无法执行文件写入，即使被注入恶意代码
  ]
}
```

Tauri v2 的细粒度权限声明，比 v1 的 allowlist 更精确。

### 外部链接安全

```typescript
// markdown.ts — 外部链接加 rel=noreferrer，防止 opener 攻击
tokens[idx].attrSet("rel", "noreferrer noopener");
```

---

## 15. 工程约定与边界

### 强制约定（违反会导致 bug）

| 约定 | 原因 |
|---|---|
| invoke() 只走 `lib/tauri.ts` | 接口变更只改一处，保持类型一致 |
| markdown-it 只有 `lib/markdown.ts` 一个实例 | 多实例会有插件状态不一致 |
| Shiki 是懒加载单例 | 加载 grammars 很贵，必须只做一次 |
| IPC 路径全部绝对路径 | 避免工作目录歧义 |
| 文件夹状态只在 `registry.rs` | 不散落，并发访问安全 |
| Marky 永远不写入被监听的文件夹 | 只读查看器，不污染用户数据 |

### shadcn 使用规范

```bash
# 添加新组件
pnpm dlx shadcn@latest add <component>

# 添加后文件在 src/components/ui/ 里，完全属于项目，可自由修改
# 不要在已有组件上重复 add（会覆盖你的修改）
```

### 新增 Markdown 功能的标准流程

1. 找 `markdown-it-*` 插件（优先于自己写 renderer rule）
2. 在 `src/lib/markdown.ts` 注册
3. 在 `src/styles/markdown.css` 添加样式
4. 在 `src/lib/__fixtures__/` 添加测试 fixture + Vitest snapshot

---

## 16. 核心设计决策与权衡

### 只读而不是编辑器

**选择**：不加 `contenteditable`，不加 `<textarea>`，完全只读。

**理由**：只读让渲染管线极简——不需要处理保存、撤销、编辑状态、冲突检测。做好一件事。

### 搜索在 Rust 侧（nucleo）

**选择**：模糊搜索在 Rust 里用 nucleo 引擎，不在 JS 里用 fuse.js。

**理由**：对大量文件（数千个 `.md` 文件的 vault），JS 模糊搜索性能差。nucleo 是 Helix 编辑器同款引擎，索引在内存里，查询延迟极低。前端只管 "输入 query，收结果"。

### `useReducer` 而不是第三方状态库

**选择**：tabs/panes 状态用 React 内置 `useReducer`。

**理由**：应用状态结构不复杂，纯 reducer 在 `workspace.ts` 里完全可测试，不引入额外依赖和学习成本。

### Mermaid 两段式处理

**选择**：解析时输出 `mermaid-pending` 标记，DOM ready 后再异步渲染。

**理由**：
- markdown-it 是同步的，不能直接 await
- 如果在同一个 async 函数里同时跑 Shiki 和 Mermaid，两者都在替换 `<pre>` 元素，会有 race condition
- 分开处理：Shiki 先完成所有代码块替换，Mermaid 再查找 `mermaid-pending` 元素

### 分栏新 Pane 留空而不是复制当前 Tab

**选择**：`SPLIT` action 创建的新 pane 是空的，不克隆当前 tab。

**理由**：测试发现两个 `Viewer` 同时对同一 source 执行高亮，会互相 race、步踩 `pre.outerHTML` 替换，导致渲染卡死。留空更安全，用户手动打开第二个文件。

---

## 17. 可复用的技巧总结

### 技巧 1：`<template>` 元素做 HTML 字符串到 DOM 节点的安全转换

```javascript
const tpl = document.createElement("template");
tpl.innerHTML = htmlString.trim();
const node = tpl.content.firstElementChild;
targetElement.replaceWith(node);
// 比 innerHTML 更安全：不执行脚本，不触发资源加载
```

### 技巧 2：异步 Effect 的取消模式

```typescript
useEffect(() => {
  let cancelled = false;
  (async () => {
    const result = await someAsyncWork();
    if (cancelled) return;  // 组件已卸载，不更新状态
    setState(result);
  })();
  return () => { cancelled = true; };
}, [deps]);
```

### 技巧 3：回调函数用 ref 避免 Effect 重跑

```typescript
// 父组件每次渲染产生新函数引用，如果放进 deps 会无限重跑
const onRenderedRef = useRef(onRendered);
useEffect(() => { onRenderedRef.current = onRendered; }, [onRendered]);
// Effect 里用 ref 读取，不加入 deps
```

### 技巧 4：`useMemo` 稳定 `dangerouslySetInnerHTML` 对象

```typescript
// 防止父组件 re-render 时 innerHTML 被重置
const dangerousHtml = useMemo(() => ({ __html: html }), [html]);
<article dangerouslySetInnerHTML={dangerousHtml} />
```

### 技巧 5：模块级 Map 做跨渲染周期的持久存储

```typescript
// 在组件外部定义，不会因组件卸载而丢失
const scrollMemory = new Map<string, number>();

// 在组件内使用
const saved = scrollMemory.get(filePath) ?? 0;
el.scrollTop = saved;
```

### 技巧 6：`nonce` 模式触发子组件重新扫描

```typescript
// 当无法通过 props 传递"内容已更新"的信号时
// 用一个递增计数器作为依赖项触发 Effect 重跑

const [contentNonce, setContentNonce] = useState(0);
// 某个异步操作完成后
setContentNonce(n => n + 1);
// 子组件监听
useEffect(() => { /* 重新扫描 */ }, [contentNonce]);
```

### 技巧 7：被动滚动监听（passive: true）

```typescript
el.addEventListener("scroll", onScroll, { passive: true });
// passive: true 告诉浏览器不会调用 preventDefault()
// 浏览器可以直接滚动而不用等 JS 执行，性能更好
```

### 技巧 8：Tauri 全局键盘快捷键的清理

```typescript
useEffect(() => {
  const onKey = (e: KeyboardEvent) => { /* ... */ };
  window.addEventListener("keydown", onKey);
  return () => window.removeEventListener("keydown", onKey);
  // deps 里放快捷键处理需要的状态，确保闭包不过时
}, [state.activePaneId, activePane.id]);
```

### 技巧 9：树遍历时用手动递归替代 TreeWalker

```typescript
// happy-dom（测试环境）的 TreeWalker 对 NodeFilter 支持不完整
// 用手动递归代替，测试环境和浏览器环境行为一致

const collect = (node: Node) => {
  if (node.nodeType === Node.TEXT_NODE) { /* 处理 */; return; }
  if (node.nodeType !== Node.ELEMENT_NODE) return;
  for (const child of Array.from(node.childNodes)) collect(child);
};
```

### 技巧 10：Rust release 构建的体积优化

```toml
[profile.release]
panic = "abort"    # 不展开 panic 栈，减少代码体积
codegen-units = 1  # 允许跨 crate 优化
lto = true         # 链接时优化，消除死代码
opt-level = "s"    # 优化体积（"z" 更激进但可能更慢）
strip = true       # 去掉 DWARF 调试信息
```

---

*笔记整理时间：2026-04-18*
*源代码库：Marky — Tauri v2 桌面 Markdown 查看器*
