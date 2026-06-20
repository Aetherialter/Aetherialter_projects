# ClickThroughPolicy 穿透策略保护记录

## 结论

本次没有继续默认启用整窗鼠标穿透，而是新增 `ClickThroughPolicy` 策略保护。

原因很直接：

如果 collapsed 状态立刻启用整窗 `WS_EX_TRANSPARENT`，WebView 很可能再也收不到 hover 事件。结果就是岛看起来“很轻量”，但用户鼠标移上去也唤不醒它。

因此当前默认策略改成：

```text
InteractiveOnly
```

含义是：

- Rust 仍然接收 shell state。
- Win32 封装仍然保留完整穿透能力。
- 默认不把窗口切到整窗穿透。
- 避免 hover/collapse 交互死锁。

## 修改范围

```text
D:\Aether_island\src-tauri\src\lib.rs
```

## 新增策略

```rust
enum ClickThroughPolicy {
    InteractiveOnly,
    FullWindowWhenCollapsed,
}
```

### InteractiveOnly

默认策略。

无论当前是：

```text
hovered
collapsed
```

都保持窗口可交互。

这个策略牺牲了一部分“零侵入穿透感”，但保证当前版本不会因为 Win32 样式切换导致交互失效。

### FullWindowWhenCollapsed

调试策略。

当 shell state 是：

```text
collapsed
```

时启用：

```text
WS_EX_TRANSPARENT
```

当 shell state 是：

```text
hovered
```

时移除：

```text
WS_EX_TRANSPARENT
```

这个策略保留给后续真实行为测试和底层能力验证。

## 新增 Tauri command

新增：

```rust
set_click_through_policy(policy: &str, ...)
```

支持字符串：

```text
interactive-only
full-window-when-collapsed
```

当前前端还没有设置入口，但 Rust 侧协议已预留好。后续可以由：

- 调试按钮。
- 托盘菜单。
- 配置文件。
- 开发者命令面板。

来切换策略。

## 为什么记录 last_shell_state

新增：

```rust
last_shell_state: Mutex<ShellState>
```

原因是策略切换时，需要知道当前窗口到底处于：

```text
hovered
collapsed
```

如果不知道当前状态，只能假设 collapsed，这会导致策略切换行为不准确。

## 小白理解

现在系统里有三层：

```text
ShellState
  表示岛当前是展开还是折叠

ClickThroughPolicy
  表示是否允许折叠时整窗穿透

window_utils
  真正调用 Win32 API 修改窗口样式
```

也就是说：

`ShellState` 是“发生了什么”；  
`ClickThroughPolicy` 是“我们允许怎么处理”；  
`window_utils` 是“真正动 Windows 窗口”。

## 为什么这是必要的架构保护

Aether_island 的目标不是单纯“把窗口穿透打开”。

真正目标是：

- 平时尽量不影响用户操作。
- 用户需要时能顺滑唤醒。
- 展开后能交互。
- 折叠后尽量低侵入。

整窗 `WS_EX_TRANSPARENT` 只能满足其中一部分。它不能区分胶囊区域和透明空白区域。

所以长期方案应当是区域级命中测试。

## 下一步方向

更合理的 Win32 方案是：

```text
WM_NCHITTEST
```

也就是让 Windows 询问：

```text
这个鼠标位置应该被窗口接收吗？
```

未来可以做到：

- 透明空白区域返回穿透。
- 胶囊实体区域保持可交互。
- 展开区域保持可交互。
- 折叠区域仍保留 hover 唤醒能力。

这比整窗 `WS_EX_TRANSPARENT` 更符合动态岛需求。

## 验证结果

以下命令已通过：

```powershell
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
pnpm build
```

## 仍需人工验收

运行：

```powershell
pnpm tauri dev
```

检查默认策略下：

- 岛是否仍能 hover 展开。
- 启动时是否不会因为 collapsed 立刻失去交互。
- 置顶、透明、动画是否仍正常。
