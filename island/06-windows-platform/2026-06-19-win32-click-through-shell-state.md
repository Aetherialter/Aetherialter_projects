# Win32 鼠标穿透接入 ShellState 记录

## 结论

本次新增 Windows 原生窗口控制层，把前端回传的 shell state 接入 Win32 扩展窗口样式。

当前行为目标：

```text
collapsed
  -> 启用 WS_EX_TRANSPARENT
  -> 鼠标点击穿透窗口

hovered
  -> 移除 WS_EX_TRANSPARENT
  -> 胶囊恢复可交互
```

同时启动时注入：

```text
WS_EX_TOOLWINDOW
WS_EX_TOPMOST
```

让窗口更接近系统级悬浮工具窗。

## 修改范围

```text
D:\Aether_island\src-tauri\Cargo.toml
D:\Aether_island\src-tauri\src\window_utils.rs
D:\Aether_island\src-tauri\src\lib.rs
```

## 新增依赖

```toml
raw-window-handle = "0.6"
windows-sys = { version = "0.59", features = ["Win32_Foundation", "Win32_UI_WindowsAndMessaging"] }
```

### 为什么使用 windows-sys

`windows-sys` 是低层 Windows API 绑定。

特点：

- 更接近 C API。
- 封装少。
- 运行时开销低。
- 需要自己处理 unsafe 边界和错误。

这符合 Aether_island 当前的轻量化方向。

## Tauri 到 HWND 的路径

Tauri v2.11 的窗口类型实现了：

```text
raw_window_handle::HasWindowHandle
```

当前代码通过：

```rust
window.window_handle()?.as_raw()
```

拿到：

```rust
RawWindowHandle::Win32(handle)
```

再从中取出：

```rust
handle.hwnd
```

这个 `HWND` 就是 Win32 API 操作窗口的原生句柄。

## window_utils.rs 职责

新增文件：

```text
D:\Aether_island\src-tauri\src\window_utils.rs
```

当前提供两个安全入口：

```rust
apply_shell_window_style(...)
set_click_through(...)
```

外部模块不直接调用 Win32 API，只拿 `Result<(), String>`。

## unsafe 边界

当前 unsafe 被限制在 `window_utils.rs` 内部。

用到的 Win32 API：

```text
GetWindowLongPtrW
SetWindowLongPtrW
SetWindowPos
```

### GetWindowLongPtrW

用于读取窗口扩展样式：

```text
GWL_EXSTYLE
```

### SetWindowLongPtrW

用于写回新的扩展样式。

当前会设置或移除：

```text
WS_EX_TRANSPARENT
WS_EX_LAYERED
WS_EX_TOOLWINDOW
WS_EX_TOPMOST
```

### SetWindowPos

用于刷新窗口样式和设置 topmost。

写完扩展样式后调用它，是为了让 Windows 立即重新计算窗口行为。

## Rust 安全模型说明

小白理解：

- Rust 自己不知道 Win32 API 是否安全。
- 所以调用这些 C 风格系统函数必须写 `unsafe`。
- `unsafe` 不是“这里一定危险”，而是“这里由程序员保证前提正确”。

本次保证的前提：

- `HWND` 来自 Tauri 当前活着的主窗口。
- 没有跨 FFI 转移内存所有权。
- 只改窗口样式 bit，不直接读写用户内存。
- unsafe 被限制在一个文件里，外部通过安全函数调用。

## 和 ShellState 的连接

`set_shell_state` command 现在会先切换 Win32 样式：

```text
ShellState::Collapsed
  -> ClickThroughMode::Enabled

ShellState::Hovered
  -> ClickThroughMode::Disabled
```

然后继续通知 Rust runtime：

```text
NativeRuntime::handle_shell_state
```

这样窗口行为和 runtime 状态走同一个入口。

## 当前验证结果

以下命令已通过：

```powershell
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
pnpm build
```

## 仍需人工验收

编译和构建只能证明 API、类型和链接正确。

鼠标穿透属于真实窗口行为，还需要运行：

```powershell
pnpm tauri dev
```

手动检查：

- 胶囊折叠时，点击是否能穿透到后面的窗口。
- 鼠标移入胶囊时，是否恢复 hover 展开。
- hover 状态下点击是否仍能被窗口接收。
- topmost 是否稳定保持。

## 当前风险

存在一个交互设计风险：

如果 collapsed 状态完全启用鼠标穿透，那么鼠标 hover 事件可能不再进入 WebView，导致无法通过 hover 自动恢复交互。

后续可能需要改成：

- 只在非胶囊区域穿透。
- 使用更细的命中测试。
- 通过 Win32 `WM_NCHITTEST` 返回 `HTTRANSPARENT` 控制区域级穿透。
- 或者保留一条极小交互热区。

这是下一轮需要重点观察和修正的地方。
