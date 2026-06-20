# Window Style Rule Tests

日期：2026-06-19

## 结论

本轮把 `window_utils.rs` 里的 Win32 extended style bit 运算抽成纯函数，并补了单元测试。

现在窗口样式规则不只靠人工探针验证，也有 `cargo test` 回归保护。

Rust 单元测试数量从 12 个增加到 16 个。

## 为什么要抽纯函数

原来的逻辑直接写在 Win32 操作函数里：

```rust
let next_styles = ...
set_extended_styles(hwnd, next_styles)?;
```

这样能跑，但测试困难，因为需要真实窗口句柄 `HWND`。

本轮抽出两个纯函数：

```rust
fn shell_extended_styles(styles: isize) -> isize
```

负责：

- 加 `WS_EX_TOOLWINDOW`
- 加 `WS_EX_TOPMOST`
- 加 `WS_EX_NOACTIVATE`
- 移除 `WS_EX_APPWINDOW`

```rust
fn click_through_extended_styles(styles: isize, mode: ClickThroughMode) -> isize
```

负责：

- 启用穿透时加 `WS_EX_LAYERED | WS_EX_TRANSPARENT`
- 禁用穿透时保留 `WS_EX_LAYERED`，移除 `WS_EX_TRANSPARENT`

这样测试不需要打开真实窗口，只需要输入一个 bit mask，检查输出 bit mask。

## 小白解释

Win32 的窗口样式可以理解成一排开关：

```text
是否置顶
是否工具窗口
是否透明层
是否鼠标穿透
是否普通应用窗口
是否不抢焦点
```

每个开关都是一个 bit。

这次测试做的事情就是：

```text
给一组开关输入，看输出时哪些开关被打开、哪些开关被关闭。
```

比起每次都手动打开窗口看行为，单元测试更适合保护这些基础规则。

## 新增测试

### 1. shell 样式添加核心标志并移除 AppWindow

测试：

```rust
shell_extended_styles_adds_shell_flags_and_removes_appwindow
```

验证：

- `WS_EX_TOOLWINDOW = true`
- `WS_EX_TOPMOST = true`
- `WS_EX_NOACTIVATE = true`
- `WS_EX_APPWINDOW = false`

### 2. shell 样式保留无关标志

测试：

```rust
shell_extended_styles_preserves_unrelated_flags
```

验证已有的 `WS_EX_LAYERED` 不会被误删。

### 3. 启用 click-through

测试：

```rust
click_through_enabled_adds_layered_and_transparent
```

验证启用穿透时：

- `WS_EX_LAYERED = true`
- `WS_EX_TRANSPARENT = true`

### 4. 禁用 click-through

测试：

```rust
click_through_disabled_keeps_layered_and_removes_transparent
```

验证禁用穿透时：

- `WS_EX_LAYERED = true`
- `WS_EX_TRANSPARENT = false`

## Rust 设计点

这次没有新增 unsafe。

unsafe 仍然只存在于真正调用 Win32 API 的地方：

- `GetWindowLongPtrW`
- `SetWindowLongPtrW`
- `SetWindowPos`

新增的样式组合函数是纯 Rust bit 运算，可测、可读、没有 FFI 风险。

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
- Rust 单元测试 16 个全部通过。
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

进程检查未发现项目 `aether-island` 或 Vite 残留。

## 对 v0.4 的意义

v0.4 的窗口层现在有三层证据：

1. Tauri 配置表达窗口意图。
2. Win32 探针证明真实窗口样式。
3. Rust 单元测试保护样式组合规则。

这让后续继续调整鼠标穿透或窗口策略时，不容易把核心浮窗属性改坏。
