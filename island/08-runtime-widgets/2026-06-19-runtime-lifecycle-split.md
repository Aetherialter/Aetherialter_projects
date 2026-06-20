# Runtime 生命周期拆分记录

## 结论

本次修正了一个重要的运行时设计问题：

旧逻辑在 `boot_events()` 里同时做了：

```text
启动刷新
广播 ShutdownRequested
shutdown_all
```

这对未来长期运行的 widget runtime 是错误的。

现在已经拆成：

```text
boot_events()
  只做启动阶段状态广播和首次刷新

shutdown_events()
  只在应用退出时广播 shutdown 并关闭 widget
```

## 修改范围

```text
D:\Aether_island\src-tauri\src\core\runtime.rs
D:\Aether_island\src-tauri\src\lib.rs
```

## 为什么要改

如果启动时就调用：

```rust
shutdown_all()
```

那么后续真正接入：

- GitHub widget
- Pomodoro widget
- Clipboard widget
- Python sidecar widget

时，它们可能在 app 刚启动后就被关闭。

这会让 runtime 看起来能发一次 mock snapshot，但不能真正长期运行。

## 新生命周期

### 启动阶段

```rust
NativeRuntime::boot_events()
```

当前做：

```text
ShellEvent::Collapsed
refresh_all
```

含义是：

- runtime 默认进入 collapsed 状态。
- 执行一次首次刷新。
- 把刷新结果推给前端。

### 退出阶段

```rust
NativeRuntime::shutdown_events()
```

当前做：

```text
ShellEvent::ShutdownRequested
shutdown_all
```

含义是：

- 通知 widget 应用即将退出。
- 给 widget 释放资源的机会。
- 后续 Python sidecar 可以在这里终止子进程。

## Tauri 退出生命周期接入

原来使用：

```rust
.run(tauri::generate_context!())
```

现在改为：

```rust
let app = tauri::Builder::default()
    ...
    .build(tauri::generate_context!())?;

app.run(|app_handle, event| {
    if let RunEvent::Exit = event {
        shutdown_runtime(app_handle);
    }
});
```

这样可以在 Tauri 应用退出事件里执行 runtime 清理。

## Rust 所有权与状态解释

`RuntimeState` 由 Tauri `manage` 托管：

```rust
RuntimeState {
    runtime: Mutex<NativeRuntime>,
    ...
}
```

退出时通过：

```rust
app_handle.state::<RuntimeState>()
```

重新取到同一个 runtime，然后加锁调用：

```rust
runtime.shutdown_events()
```

这保证启动、交互、退出都操作同一个 runtime 实例。

## 对 v0.4 的意义

这一步为 sidecar-ready 做了必要铺垫。

未来如果有：

```text
PythonSidecarWidget
```

它可以在：

```text
boot_events
```

里启动或首次刷新，在：

```text
shutdown_events
```

里优雅终止 Python 子进程。

否则双进程很容易出现僵尸进程或资源泄漏。

## 验证结果

以下命令已通过：

```powershell
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
pnpm build
```

## 下一步建议

下一步适合新增周期刷新骨架：

```text
NativeRuntime::refresh_events()
```

然后由后台线程或轻量 scheduler 定期调用。

这样 runtime 就从“启动时刷新一次”进化到“可长期驱动 widget”。
