# 前端 ShellState 回传 Rust Runtime 记录

## 结论

本次打通了前端 hover/collapse 状态到 Rust runtime 的通道。

现在胶囊岛的交互状态不再只是 CSS class，而是会通过 Tauri command 同步给 Rust：

```text
mouseenter / mouseleave
  -> TypeScript setIslandState
  -> invoke("set_shell_state")
  -> Rust set_shell_state command
  -> NativeRuntime::handle_shell_state
  -> NativeWidgetProvider::broadcast_shell_event
  -> IslandWidget::handle_shell_event
```

这一步为后续 Win32 鼠标穿透、点击响应、组件暂停刷新、展开态数据加载打基础。

## 修改范围

```text
D:\Aether_island\src\main.ts
D:\Aether_island\src-tauri\src\lib.rs
D:\Aether_island\src-tauri\src\core\runtime.rs
```

## 前端变化

前端新增：

```ts
import { invoke } from "@tauri-apps/api/core";
```

在状态变化时调用：

```ts
await invoke("set_shell_state", { state });
```

其中 `state` 当前只允许：

```text
hovered
collapsed
```

前端失败时只记录 warning，不阻断 UI 动画。这样 Rust 通道临时失败时，视觉交互仍能继续。

## Rust command

Rust 新增 Tauri command：

```rust
#[tauri::command]
fn set_shell_state(...)
```

它负责：

- 校验前端传来的字符串。
- 获取全局 runtime。
- 加锁拿到可变 runtime。
- 调用 `NativeRuntime::handle_shell_state`。
- 把可能产生的前端事件 emit 回 WebView。

## 为什么使用 Builder::manage

如果每次 command 都创建一个新的 `NativeRuntime`，那只是“假同步”。

真实状态必须长期存在，所以当前用：

```rust
.manage(RuntimeState {
    runtime: Mutex::new(NativeRuntime::new()),
})
```

小白理解：

- `manage`：把一个对象交给 Tauri 托管，整个应用运行期间都能取到它。
- `RuntimeState`：应用级状态盒子。
- `Mutex`：给 runtime 加锁，避免多个命令同时修改它。

## Rust 生命周期踩坑

一开始不能把：

```rust
State<'_, RuntimeState>
```

直接移动进 `thread::spawn`。

原因是 `State<'_>` 是短生命周期借用，只保证当前函数里有效；而 `thread::spawn` 要求闭包里的数据能活到线程结束，也就是近似 `'static`。

正确做法是把 `AppHandle` 移进线程，然后在线程里重新取 state：

```rust
let runtime_state = app_handle.state::<RuntimeState>();
```

这体现了 Rust 的核心规则：

生命周期标注不会延长数据寿命，只会描述数据之间的借用关系。

## MutexGuard 析构顺序问题

编译器还提示过 `runtime_state` 不够长。

根因是：

```rust
runtime_state.runtime.lock()
```

会产生一个临时 `Result<MutexGuard<...>>`。如果临时值析构顺序不清晰，借用检查器会认为锁的借用可能活得太久。

最终写法使用：

```rust
if let Ok(mut runtime) = runtime_state.runtime.lock() {
    ...
};
```

这个末尾分号让临时值更早释放，同时满足 clippy。

## 当前状态

已经完成：

- 前端状态回传 Rust。
- Rust command 参数校验。
- Tauri managed runtime。
- Mutex 保护 runtime 可变访问。
- runtime 将 shell state 广播给 provider。

还没有完成：

- 从 shell state 反向控制 Win32 鼠标穿透。
- 多窗口状态同步。
- 周期刷新调度。
- command 级测试。

## 验证结果

以下命令已通过：

```powershell
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
pnpm build
```

## 下一步建议

下一步适合进入 Windows 原生层：

```text
ShellState::Collapsed
  -> 启用鼠标穿透

ShellState::Hovered
  -> 关闭鼠标穿透
```

这会用到 Win32 扩展窗口样式：

```text
WS_EX_TRANSPARENT
WS_EX_LAYERED
```

建议先封装 `window_utils.rs`，再接入 runtime shell state。
