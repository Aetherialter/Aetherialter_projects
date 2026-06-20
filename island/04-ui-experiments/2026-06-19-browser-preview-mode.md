# Browser Preview Mode

日期：2026-06-19

## 结论

本轮为前端增加了普通浏览器预览模式。目的不是替代 Tauri 桌面验收，而是让 UI 布局、hover 展开、按钮区域和文本切换可以在浏览器里稳定验证。

当前浏览器预览已验证：

- 折叠态岛水平居中。
- 折叠态视觉尺寸约 `213x32`，对应 CSS `292x44` 经 `scale(0.73)` 缩放后的结果。
- 展开态尺寸为 `292x44`。
- 预览 widgets 显示 `Clock / Mock / Sidecar`。
- 点击展开后，点击 `Mock` 可以切换标题、描述和 active 按钮。
- 已保存截图：`D:\Users\Aetherialter\Documents\Aetherialter_projects\island\04-ui-experiments\2026-06-19-browser-preview-hovered.png`

## 为什么需要浏览器预览模式

普通浏览器没有 Tauri runtime，所以直接打开 Vite 页面时，下列 API 会失败：

- `invoke`
- `listen`
- `getCurrentWindow`
- `primaryMonitor`

这些 API 只能在 Tauri WebView 容器里正常工作。之前浏览器控制台会出现 Tauri API 相关警告，并且 widget 列表不能加载。

本轮新增 `isTauriRuntime()` 判断：

```ts
function isTauriRuntime(): boolean {
  return "__TAURI_INTERNALS__" in window;
}
```

如果不在 Tauri 环境中：

- 不调用 Rust command。
- 不监听 Tauri event。
- 不调用 Tauri window API。
- 使用本地 `previewWidgets` 渲染 `Clock / Mock / Sidecar`。
- 点击 widget 时本地渲染 preview snapshot。

如果在 Tauri 环境中，仍然走原来的 Rust runtime 和 IPC 链路。

## 浏览器验收数据

折叠态测量：

```json
{
  "centerDelta": 0,
  "island": {
    "width": 213.16,
    "height": 32.12,
    "y": 18
  },
  "opacity": "0.86",
  "transform": "matrix(0.73, 0, 0, 0.73, 0, 0)"
}
```

展开态测量：

```json
{
  "centerDelta": 0,
  "className": "island island--hovered",
  "island": {
    "width": 292,
    "height": 44,
    "y": 18
  },
  "activeText": "Mock",
  "activeWidgetId": "mock",
  "title": "Mock",
  "subtitle": "mock preview"
}
```

## 本轮改动

### `src\main.ts`

新增：

- `previewWidgets`
- `isTauriRuntime()`
- `renderPreviewSnapshot()`

调整：

- 非 Tauri 环境下 `notifyShellState()` 直接返回。
- 非 Tauri 环境下 `positionWindow()` 直接返回。
- 非 Tauri 环境下 `loadWidgets()` 使用本地预览 widgets。
- 非 Tauri 环境下 `setActiveWidget()` 直接渲染本地 preview snapshot。
- 非 Tauri 环境下点击岛本体也能展开，方便浏览器自动化测试。

## 验证命令

在 `D:\Aether_island` 运行：

```powershell
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo test --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
pnpm build
```

结果：

- Rust 格式检查通过。
- Rust 单元测试 12 个全部通过。
- Clippy 以 `-D warnings` 通过。
- 前端 TypeScript 与 Vite production build 通过。

Python sidecar 回归：

```powershell
uv run ruff format .
uv run ruff check .
uv run pytest
```

结果：

- ruff format 通过。
- ruff check 通过。
- pytest 6 个测试全部通过。

## 未覆盖边界

浏览器预览不能证明：

- Tauri 透明窗口效果。
- Win32 置顶行为。
- 任务栏隐藏。
- 鼠标穿透。
- Tauri WebView 里的真实 IPC 行为。

这些仍需要 `pnpm tauri dev` 下的桌面窗口验收。

## 对 v0.4 的意义

v0.4 的 UI 现在有了一个轻量、可重复的预览入口。以后修改视觉、动画、按钮布局时，可以先用浏览器预览验证，再进入 Tauri 桌面窗口验收。
