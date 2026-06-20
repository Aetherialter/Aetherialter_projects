# SystemClockWidget 本地时钟组件记录

## 结论

本次新增第一个真实本地 widget：

```text
SystemClockWidget
```

它不依赖网络、不依赖 Python、不需要 token，通过 Windows 本地时间 API 读取系统时间，并每秒刷新到前端。

前端显示示例：

```text
local 21:35:08
```

这一步让 Aether_island 从 mock 计数推进到真实系统数据源。

## 修改范围

```text
D:\Aether_island\src-tauri\Cargo.toml
D:\Aether_island\src-tauri\src\core\widget.rs
D:\Aether_island\src-tauri\src\core\provider.rs
```

## 新增依赖 feature

`windows-sys` 新增：

```toml
"Win32_System_SystemInformation"
```

用于调用：

```rust
GetLocalTime
```

## 为什么不用 chrono

当前只需要本地系统时间的小时、分钟、秒。

如果引入 `chrono`，会增加额外依赖和构建复杂度。

当前项目是 Windows-first、轻量化方向，所以直接使用：

```text
windows-sys
```

更符合底层性能和零侵入目标。

## SystemClockWidget 行为

实现：

```rust
pub struct SystemClockWidget;
```

刷新间隔：

```rust
Duration::from_secs(1)
```

刷新逻辑：

```text
GetLocalTime
  -> SYSTEMTIME
  -> local HH:MM:SS
  -> WidgetSnapshot::ready(...)
```

## unsafe 边界

调用：

```rust
GetLocalTime(&mut time)
```

需要 `unsafe`。

安全前提：

- `time` 是当前栈上的有效 `SYSTEMTIME` 变量。
- 传入的是可写指针。
- Windows API 只在函数调用期间写入该结构。
- API 不保存这个指针。

因此 unsafe 范围很小，并且没有跨线程或跨生命周期保存裸指针。

## Provider 注册顺序

当前注册顺序：

```text
SystemClockWidget
MockWidget
```

`SystemClockWidget` 每 1 秒刷新。

`MockWidget` 调整为每 30 秒刷新，只作为调试心跳存在。

这样 UI 大多数时间会显示系统时钟，不会被 mock 计数频繁覆盖。

## 对 v0.4 的意义

当前 runtime 已经具备：

- 多 widget 注册。
- 周期刷新。
- 本地系统数据读取。
- Rust 到前端 snapshot 推送。

这比单纯 mock 更接近真实组件系统。

后续接入 GitHub、Pomodoro、Clipboard 或 Python sidecar 时，都可以沿用同一个 `IslandWidget` trait。

## 验证结果

以下命令已通过：

```powershell
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
pnpm build
```

## 人工验收

运行：

```powershell
pnpm tauri dev
```

观察胶囊副标题是否每秒显示新的本地时间：

```text
local HH:MM:SS
```

## 下一步建议

下一步应该引入 widget 显示优先级或 active widget 选择。

否则多个 widget 同时刷新时，谁最后 emit，谁就覆盖 UI。

建议新增：

```text
WidgetSnapshot {
  plugin_id
  priority
  updated_at
}
```

或者先在 provider 层实现简单的 active widget 选择策略。
