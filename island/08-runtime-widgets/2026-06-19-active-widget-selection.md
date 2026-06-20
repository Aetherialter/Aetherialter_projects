# Active Widget 选择策略记录

## 结论

本次新增 `active_widget_id` 选择策略。

现在 runtime 不只会按静态优先级自动选择 snapshot，还可以记住“当前关注的 widget”。

选择顺序：

```text
如果 active_widget_id 对应的 widget 有 snapshot
  -> 显示 active widget
否则
  -> 回退到 priority 最高的 snapshot
```

## 修改范围

```text
D:\Aether_island\src-tauri\src\core\provider.rs
D:\Aether_island\src-tauri\src\core\runtime.rs
D:\Aether_island\src-tauri\src\lib.rs
```

## Provider 变化

新增：

```rust
contains_widget(widget_id)
```

用于校验传入的 widget id 是否存在。

`refresh_all` 从：

```rust
refresh_all()
```

变成：

```rust
refresh_all(active_widget_id: Option<&str>)
```

它会：

1. 刷新所有 widget。
2. 如果 active widget 有成功 snapshot，优先选择 active。
3. 否则选择 priority 最高的 snapshot。
4. 错误事件仍然保留。

## Runtime 变化

`NativeRuntime` 新增字段：

```rust
active_widget_id: Option<String>
```

新增方法：

```rust
set_active_widget(widget_id: Option<String>)
```

如果传入未知 id，会返回错误：

```text
unknown widget id: xxx
```

## Tauri Command

新增：

```rust
set_active_widget(widget_id: Option<String>)
```

语义：

```text
Some("system-clock") -> 关注时钟
Some("mock")         -> 关注 mock
Some("")            -> 清除关注
None                -> 清除关注
```

当前前端还没有按钮调用它，但 Rust runtime 的接口已经准备好。

## 为什么 active_widget_id 放在 Runtime

Provider 管 widget 列表和刷新。

Runtime 管运行时策略。

`active_widget_id` 是“当前用户关注谁”的运行时状态，所以放在 `NativeRuntime` 更合适。

这样未来前端、托盘、快捷键、配置文件都可以通过同一个 runtime 接口修改 active widget。

## 小白理解

之前是：

```text
谁优先级高，谁上屏
```

现在是：

```text
用户点名谁，谁先上屏
如果用户没点名，才看优先级
```

这就是从“自动排序”走向“可交互组件系统”的一步。

## 对 v0.4 的意义

后续有多个组件时，比如：

```text
Clock
GitHub
Pomodoro
Clipboard
PythonSidecar
```

用户 hover、点击、托盘菜单、快捷键都可能切换当前关注组件。

`active_widget_id` 是这个能力的核心入口。

## 验证结果

以下命令已通过：

```powershell
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
pnpm build
```

## 下一步建议

下一步可以在前端加一个极简开发态切换入口，例如：

```text
hover 展开时显示两个小段：
Clock | Mock
```

点击后调用：

```ts
invoke("set_active_widget", { widgetId: "mock" })
```

这样就能真正从 UI 层切换 active widget。
