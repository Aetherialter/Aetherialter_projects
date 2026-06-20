# Sidecar Ready 边界记录

## 结论

本次新增了 Rust 侧 sidecar-ready 边界。

当前不启动 Python 进程，不做 stdio 通信，只先定义：

- IPC 消息契约。
- Sidecar 状态结构。
- `SidecarWidget` 生命周期骨架。
- provider 注册入口。

这让项目后续可以平滑演进到 Rust + Python 双进程，而不会推翻现有 widget/runtime 架构。

## 修改范围

```text
D:\Aether_island\src-tauri\src\core\sidecar.rs
D:\Aether_island\src-tauri\src\core\mod.rs
D:\Aether_island\src-tauri\src\core\widget.rs
D:\Aether_island\src-tauri\src\core\provider.rs
```

## 新增 sidecar.rs

新增 IPC 消息类型：

```rust
SidecarMessage
```

当前包含：

```text
DAEMON_READY
RENDER_UPDATE
SHELL_STATE
SHUTDOWN
```

这些名字对齐之前规划的 Python daemon 协议。

## 消息 Payload

### SidecarReadyPayload

```rust
status
active_plugins
```

用于 Python daemon 告诉 Rust：

```text
我启动好了，当前有哪些插件
```

### SidecarRenderPayload

```rust
plugin
title
subtitle
priority
```

用于 Python daemon 推送某个插件的可渲染状态。

### SidecarShellStatePayload

```rust
state
focus_plugin
```

用于 Rust 把当前岛状态告诉 Python，例如：

```text
hovered
collapsed
shutdown
```

## SidecarWidget

新增：

```rust
SidecarWidget
```

当前它是一个低优先级 widget：

```text
id = sidecar
title = Sidecar
priority = 1
refresh_interval = 60s
```

它不会抢占 Clock 的显示。

## 当前行为

当前 `SidecarWidget` 不启动子进程，只显示：

```text
sidecar online
```

当收到 shutdown 事件时，状态变为：

```text
sidecar stopped
```

这用于验证 lifecycle 接口已经接入。

## 为什么现在不启动 Python

当前阶段先定义边界，而不是立刻做双进程。

原因：

- 避免过早引入子进程生命周期复杂度。
- 保持 Rust-first 主体稳定。
- 先验证 `IslandWidget` trait 是否能承载 sidecar 概念。
- 后续真接 Python 时，只需要替换 `SidecarWidget` 内部实现。

## 对 v0.4 的意义

v0.4 的关键不是“马上把 Python 跑起来”，而是让架构具备可演进到双进程的能力。

现在已有：

```text
Rust runtime
  -> NativeWidgetProvider
      -> SystemClockWidget
      -> MockWidget
      -> SidecarWidget
```

未来可以演进为：

```text
SidecarWidget
  -> spawn python daemon
  -> stdio JSON
  -> SidecarMessage
  -> WidgetSnapshot
```

## 验证结果

以下命令已通过：

```powershell
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
pnpm build
```

## 下一步建议

下一步可以新增一个 `sidecar/` 目录，放置 Python daemon 的最小协议草图：

```text
island-daemon/main.py
island-daemon/plugins/
```

但建议仍先不自动启动它，先用命令行手动跑通 JSON 协议。
