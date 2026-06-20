# Win32 No-Activate Window

日期：2026-06-19

## 结论

本轮给 Aether_island 主窗口加入了 `WS_EX_NOACTIVATE`，让它更接近“状态浮层/灵动岛”的行为：显示在最前，但尽量不抢走当前正在使用的软件焦点。

修复后 Win32 探针结果：

```json
{
  "Title": "Aether_island",
  "ExStyleHex": "0x8080198",
  "ToolWindow": true,
  "TopMost": true,
  "Layered": true,
  "Transparent": false,
  "AppWindow": false,
  "NoActivate": true,
  "Width": 480,
  "Height": 120
}
```

## 为什么需要 NOACTIVATE

灵动岛类窗口不是传统应用主窗口。它更像一个一直在屏幕上方显示状态的浮层。

如果它抢焦点，用户正在输入代码、打字、操作别的软件时，就可能被打断。

`WS_EX_NOACTIVATE` 的作用可以用小白术语理解为：

```text
我可以显示在前面，但不要把键盘焦点抢过来。
```

这符合 Aether_island 当前的 Zero-Footprint 目标：轻、静、少打扰。

## 本轮代码改动

文件：

```text
D:\Aether_island\src-tauri\src\window_utils.rs
```

新增导入：

```rust
WS_EX_NOACTIVATE
```

窗口样式合成改为：

```rust
let next_styles =
    (styles | WS_EX_TOOLWINDOW as isize | WS_EX_TOPMOST as isize | WS_EX_NOACTIVATE as isize)
        & !(WS_EX_APPWINDOW as isize);
```

含义：

1. 保留 Tauri 已经设置的窗口扩展样式。
2. 加上工具窗口：`WS_EX_TOOLWINDOW`。
3. 加上置顶：`WS_EX_TOPMOST`。
4. 加上不抢焦点：`WS_EX_NOACTIVATE`。
5. 移除普通应用窗口身份：`WS_EX_APPWINDOW`。

## Rust 与 unsafe 边界解释

这次没有新增新的 unsafe 块，只是把一个新的 bit flag 加入已有样式组合。

真正接触 Win32 的 unsafe 仍集中在：

- `GetWindowLongPtrW`
- `SetWindowLongPtrW`
- `SetWindowPos`

这些函数都在 `window_utils.rs` 内部封装，外部 Rust 代码只调用安全函数：

- `apply_shell_window_style`
- `set_click_through`

这是一种比较干净的边界：把不安全的 Windows FFI 控制在一个文件里，不让 unsafe 扩散到 runtime/widget/UI 代码。

## 重要边界

`NOACTIVATE` 不等于窗口不能点击。

它主要影响焦点激活行为。按钮能不能点，还取决于：

- 当前是否有 `WS_EX_TRANSPARENT`。
- WebView 内部元素是否允许 pointer events。
- 当前 shell state 是 hovered 还是 collapsed。

当前默认策略仍是 `InteractiveOnly`：

- `Transparent = false`
- `Layered = true`

所以主窗口默认保留交互能力，不启用整窗穿透。

## 验证结果

执行：

```powershell
pnpm tauri dev
```

Win32 探针确认主窗口：

- `ToolWindow = true`
- `TopMost = true`
- `Layered = true`
- `Transparent = false`
- `AppWindow = false`
- `NoActivate = true`

随后关闭 dev 会话，并运行质量门禁：

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

进程检查未发现项目 `aether-island` 或 Vite 残留。

## 对 v0.4 的意义

窗口现在更接近 v0.4 的“零侵入浮层”目标：

1. 工具窗口身份更明确。
2. 不作为普通 app window 暴露。
3. 支持透明层。
4. 保持置顶。
5. 尽量不抢焦点。

后续仍需要验证真实交互：

- 点击按钮时是否符合预期。
- hover 展开是否稳定。
- Alt-Tab 和任务栏是否真正不出现。
- 若未来启用整窗穿透，hover 恢复策略是否可靠。
