# Native Window ExStyle Probe

日期：2026-06-19

## 结论

本轮对真实 Tauri 桌面窗口做了 Win32 扩展样式探针验收，并修复了一个窗口属性冲突：主窗口已经是 `WS_EX_TOOLWINDOW`，但仍然带着 `WS_EX_APPWINDOW`。

修复后主窗口关键属性为：

```json
{
  "Title": "Aether_island",
  "ExStyleHex": "0x80198",
  "ToolWindow": true,
  "TopMost": true,
  "Layered": true,
  "Transparent": false,
  "AppWindow": false,
  "Width": 480,
  "Height": 120
}
```

这更符合当前 v0.4 的窗口目标：工具窗口、置顶、透明层支持、不作为普通 app window 暴露。

## 为什么要查 ExStyle

Tauri 配置里的：

```json
{
  "decorations": false,
  "transparent": true,
  "alwaysOnTop": true,
  "skipTaskbar": true
}
```

能表达意图，但不能直接证明 Win32 层的真实状态。

所以本轮用 PowerShell 调 Win32 API：

- `EnumWindows`
- `GetWindowThreadProcessId`
- `GetWindowTextW`
- `GetWindowLongPtrW(GWL_EXSTYLE)`
- `GetWindowRect`

读取真实窗口的扩展样式 bit flag。

## 修复前

第一次探针主窗口结果：

```json
{
  "Title": "Aether_island",
  "ExStyleHex": "0xC0198",
  "ToolWindow": true,
  "TopMost": true,
  "Layered": true,
  "Transparent": false,
  "AppWindow": true,
  "Width": 480,
  "Height": 120
}
```

问题是：`ToolWindow = true` 和 `AppWindow = true` 同时存在。

小白理解：

- `ToolWindow` 更像工具窗口、小浮窗。
- `AppWindow` 更像普通应用窗口，会更倾向出现在任务栏/Alt-Tab 等普通应用入口里。

两者同时存在会让窗口身份不够纯粹。

## 修复内容

文件：

```text
D:\Aether_island\src-tauri\src\window_utils.rs
```

在 `ensure_shell_extended_styles()` 中显式移除：

```rust
WS_EX_APPWINDOW
```

核心逻辑：

```rust
let next_styles =
    (styles | WS_EX_TOOLWINDOW as isize | WS_EX_TOPMOST as isize) & !(WS_EX_APPWINDOW as isize);
```

含义：

1. 保留原来的扩展样式。
2. 加上 `WS_EX_TOOLWINDOW`。
3. 加上 `WS_EX_TOPMOST`。
4. 移除 `WS_EX_APPWINDOW`。

## 修复后

重启 `pnpm tauri dev` 后重新探针：

```json
{
  "Hwnd": "0x4A0C10",
  "Visible": true,
  "Title": "Aether_island",
  "ExStyleHex": "0x80198",
  "ToolWindow": true,
  "TopMost": true,
  "Layered": true,
  "Transparent": false,
  "AppWindow": false,
  "Left": 613,
  "Top": 8,
  "Width": 480,
  "Height": 120
}
```

`Transparent = false` 是当前预期行为，因为现在默认策略是 `InteractiveOnly`，也就是不启用整窗鼠标穿透。这样可以避免折叠态完全穿透后 hover 无法恢复。

如果未来切到 `FullWindowWhenCollapsed` 策略，折叠态才应该看到 `Transparent = true`。

## 验证命令

在 `D:\Aether_island` 执行过：

```powershell
pnpm tauri dev
```

并用 Win32 探针读取真实窗口样式。

随后运行质量门禁：

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

## 进程清理

`pnpm tauri dev` 最后通过 Ctrl+C 主动停止，`STATUS_CONTROL_C_EXIT` 是手动中断导致。

后续检查没有项目 `aether-island` 或 Vite dev server 残留。剩余的 `node.exe` 是 Codex 浏览器自动化运行时，不属于项目进程。

## 对 v0.4 的意义

这一步把 v0.4 从“配置上像无头浮窗”推进到“Win32 层能证明窗口身份正确”。

当前已验证：

1. 主窗口可见。
2. 主窗口尺寸为 `480x120`。
3. 主窗口是 tool window。
4. 主窗口是 topmost。
5. 主窗口具备 layered 透明层能力。
6. 主窗口不再带 app window 扩展样式。

剩余需要人工或更深自动化验证：

1. 任务栏是否实际不显示。
2. Alt-Tab 是否实际不显示。
3. 鼠标穿透策略在交互切换中是否符合预期。
4. UI hover 和 widget 切换在真实 Tauri WebView 中是否完全符合预期。
