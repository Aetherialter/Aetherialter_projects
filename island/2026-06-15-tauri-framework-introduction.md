# Tauri 框架入门：IslandGate 为什么可以用它

## 结论

Tauri 是一个用来构建桌面应用的框架。它的核心思想是：

```text
前端界面用 HTML / CSS / JavaScript 写
系统能力和后端逻辑用 Rust 写
界面运行在系统自带 WebView 里
```

对 IslandGate 来说，Tauri 适合承担“桌面窗口壳”的角色：它能让我们用 Web 技术快速做灵动岛 UI，同时让 Rust 负责窗口、托盘、Win32、缓存、调度和后续小组件核心逻辑。

## 背景

桌面应用通常有几类写法：

- 原生 GUI：例如 Win32、WPF、Qt、GTK。
- Web 桌面壳：例如 Electron、Tauri。
- 游戏/图形框架：例如 egui、iced、slint、skia 相关方案。

Tauri 属于 Web 桌面壳，但它和 Electron 的关键区别是：

```text
Electron 自带 Chromium 和 Node.js
Tauri 默认使用操作系统已有的 WebView
```

所以 Tauri 通常更轻。Windows 上主要依赖 WebView2，macOS 上用 WKWebView，Linux 上常见是 WebKitGTK。

## 核心机制

Tauri 应用大体分成两层：

```text
WebView Frontend
    ↑↓
Tauri IPC / Commands / Events
    ↑↓
Rust Backend
```

### 前端层

前端层负责 UI：

- HTML 负责结构。
- CSS 负责样式和动画。
- JavaScript 负责交互。

它可以是原生 HTML/CSS/JS，也可以是 React、Vue、Svelte 等框架。IslandGate 第一版为了轻量和可控，推荐先使用原生 HTML/CSS/JS。

### Rust 后端层

Rust 层负责可信系统能力：

- 创建和管理窗口。
- 系统托盘。
- 文件读写。
- 本地缓存。
- 配置解析。
- 网络请求。
- 调度小组件。
- 调用平台 API，例如 Windows Win32。

前端不能随便直接碰系统资源，需要通过 Tauri 暴露的命令或事件和 Rust 通信。

### IPC 通信

前端和 Rust 之间不是直接共享内存，而是通过消息通信。

常见方向：

```text
前端 -> Rust：调用 command
Rust -> 前端：emit event
```

在 IslandGate 里可以这样理解：

```text
前端告诉 Rust：鼠标悬停了
Rust 告诉前端：GitHub Widget 有新快照
```

## Rust 知识点

### 1. Rust 在 Tauri 里是“可信核心”

Tauri 项目里的 Rust 代码不是普通脚本，而是编译后的原生二进制程序。它拥有更强的系统能力，也要承担更严格的错误处理责任。

因此 Rust 层不应该大量使用 `unwrap()` 处理外部输入。外部输入包括：

- 配置文件。
- 缓存文件。
- 网络响应。
- 前端传来的参数。
- Windows API 返回值。

更稳的方式是返回 `Result`，把错误变成可诊断状态。

### 2. 所有权适合管理资源生命周期

IslandGate 会涉及很多资源：

- 窗口句柄。
- 托盘图标。
- 异步任务。
- 缓存数据。
- 网络客户端。
- 后续可能的 Python sidecar 子进程。

Rust 的所有权模型适合表达“谁负责释放资源”。

例如：

```text
AppState 拥有 Runtime
Runtime 拥有 WidgetProvider
WidgetProvider 拥有具体 Widget
```

这样程序退出时可以从外到内有序 shutdown。

### 3. async 适合调度小组件

GitHub 刷新、缓存写入、未来脚本监控都不应该阻塞 UI。Rust 侧可以用异步 runtime 调度这些任务。

关键原则是：

```text
UI 线程不要做耗时操作
Widget 刷新失败不要拖垮整个应用
```

## Windows / 系统层知识点

Tauri 能帮我们创建窗口，但 IslandGate 需要更底层的 Windows 行为：

- 鼠标穿透。
- 隐藏任务栏图标。
- 工具窗口样式。
- 置顶窗口。
- 半透明窗口。

这些通常需要 Win32 API，例如通过窗口 handle 修改扩展窗口样式。

这也是为什么 IslandGate 不能只停留在普通 Tauri 应用层面，而要加一层 Windows platform adapter：

```text
Tauri window
↓
raw window handle
↓
windows-sys 调用 Win32
↓
修改窗口样式
```

## 工程取舍

### 为什么不用纯 Win32

纯 Win32 可以最轻，但 UI 开发成本高，动画和布局不方便。IslandGate 需要流体动画和视觉效果，用 HTML/CSS 会快很多。

### 为什么不用 Electron

Electron 生态强，但对 IslandGate 这种小型常驻组件来说太重。它会带上 Chromium 和 Node.js，内存和包体通常更大。

### 为什么用 Tauri

Tauri 的优势刚好贴合 IslandGate：

- UI 可以用 Web 技术快速做。
- Rust 可以处理底层系统能力。
- 包体和内存通常比 Electron 更轻。
- 可以保留跨平台潜力。
- 适合 Windows-first 但 cross-platform-ready 的设计。

## 验证方式

初始化 Tauri 后，先验证壳子能跑：

```powershell
npm run tauri dev
```

Rust 侧质量门禁：

```powershell
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
```

前端构建验证：

```powershell
npm run build
```

## 常见坑

### 1. 以为 Tauri 等于浏览器

Tauri 不是把网页直接丢进浏览器。它是一个原生应用壳，里面嵌入系统 WebView。前端看起来像网页，但运行边界和权限模型不同。

### 2. 把所有逻辑都写在前端

这会破坏架构。IslandGate 的数据、缓存、系统调用和调度应该放在 Rust。前端只做展示和交互事件。

### 3. 过早引入复杂前端框架

IslandGate 第一版 UI 很小，用原生 HTML/CSS/JS 更容易保持轻量。等 UI 状态复杂后再考虑框架。

### 4. 把 Win32 代码散落到业务模块

Win32 是平台细节，应该被隔离在 `platform/windows` 或类似模块里。这样以后迁移 macOS/Linux 时不会污染 core。

## 简历与面试表达

可以这样描述 IslandGate 中使用 Tauri 的价值：

```text
项目采用 Tauri 构建轻量 Windows 桌面端应用，使用系统 WebView 承载 HTML/CSS/JS 动画界面，以 Rust 作为可信后端管理窗口生命周期、系统托盘、缓存、异步调度和 Win32 平台能力。相比 Electron 方案，该设计降低了运行时开销，并通过平台抽象层保留后续跨平台迁移空间。
```

更技术化一点：

```text
在架构上将 WebView UI、Rust runtime、PlatformShell 和 WidgetProvider 解耦。UI 只消费状态快照，Rust 负责系统能力与状态调度，Win32 调用被限制在平台适配层，从而兼顾轻量化、可测试性和后续 sidecar 扩展能力。
```

## 参考来源

- Tauri v2 官方介绍：https://v2.tauri.app/start/
- Tauri v2 架构说明：https://v2.tauri.app/concept/architecture/
- Tauri GitHub 仓库：https://github.com/tauri-apps/tauri
