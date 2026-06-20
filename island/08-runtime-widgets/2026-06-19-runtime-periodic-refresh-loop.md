# Runtime 周期刷新线程记录

## 结论

本次把 Aether_island 的 runtime 从“启动时刷新一次”推进到“后台周期刷新”。

当前刷新链路：

```text
spawn_refresh_loop
  -> sleep(refresh_interval)
  -> NativeRuntime::refresh_events()
  -> NativeWidgetProvider::refresh_all()
  -> IslandWidget::refresh()
  -> FrontendEvent::Snapshot
  -> island://snapshot
  -> frontend renderSnapshot
```

这一步让 widget runtime 开始具备长期运行能力。

## 修改范围

```text
D:\Aether_island\src-tauri\src\core\provider.rs
D:\Aether_island\src-tauri\src\core\runtime.rs
D:\Aether_island\src-tauri\src\lib.rs
```

## Provider 变化

`NativeWidgetProvider` 原来提供：

```rust
refresh_intervals_ms() -> Vec<u128>
```

现在改为：

```rust
refresh_intervals() -> Vec<Duration>
```

原因是 Rust 内部调度更适合直接使用 `Duration`，不用先转成毫秒再转回来。

## Runtime 变化

新增：

```rust
NativeRuntime::refresh_interval()
```

它会取所有 widget 中最短刷新间隔。

当前 mock widget 的刷新间隔是：

```text
5 秒
```

新增：

```rust
NativeRuntime::refresh_events()
```

它只负责刷新 widget，不改变 shell state。

## 后台刷新线程

`lib.rs` 新增：

```rust
spawn_refresh_loop(app_handle)
```

它会：

1. 从 Tauri managed state 里取 runtime。
2. 读取当前刷新间隔。
3. `thread::sleep(refresh_interval)`。
4. 再次检查是否退出。
5. 调用 `runtime.refresh_events()`。
6. 把事件 emit 到前端。

## 停止信号

`RuntimeState` 新增：

```rust
stop_requested: Arc<AtomicBool>
```

小白理解：

- `AtomicBool` 是一个线程之间共享的布尔开关。
- `Arc` 允许多个线程共同持有这个开关。
- app 退出时把它设置为 `true`。
- 刷新线程看到后停止循环。

这可以避免 app 退出后刷新线程继续跑。

## 为什么暂时不用 tokio

当前需求只是一个轻量周期刷新循环。

如果现在引入 tokio，会带来：

- 新 runtime。
- 新 async 生命周期。
- 命令、锁、退出流程的复杂度。
- 更重的依赖和学习成本。

当前用标准库线程已经足够：

```rust
thread::spawn
thread::sleep
Arc<AtomicBool>
Mutex<NativeRuntime>
```

等后续真的接入网络请求、Python sidecar stdio、并发插件刷新时，再评估 async runtime。

## Rust 借用检查踩坑

周期刷新里一开始出现过 `runtime_state does not live long enough`。

原因是：

```rust
runtime_state.runtime.lock()
```

产生的临时 `Result<MutexGuard<...>>` 可能在块末尾析构，导致借用检查器认为 `runtime_state` 被借用得太久。

最终写法是：

```rust
let interval = match runtime_state.runtime.lock() {
    Ok(runtime) => runtime.refresh_interval(),
    Err(_) => Duration::from_secs(5),
};
interval
```

这样锁的临时值释放时机更清楚。

## 当前限制

当前 scheduler 是全局最短间隔刷新。

也就是说多个 widget 时：

```text
最短的那个 widget 决定全局刷新节拍
```

后续更细的设计应该是：

- 每个 widget 有自己的 next_refresh_at。
- 到期才刷新对应 widget。
- 避免低频 widget 被高频 widget 带着频繁刷新。

但当前阶段先建立长期刷新能力更重要。

## 验证结果

以下命令已通过：

```powershell
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
pnpm build
```

## 下一步建议

下一步可以做两件事之一：

1. 给 `MockWidget` 做一个可见递增计数，让前端能肉眼确认周期刷新在工作。
2. 把 refresh 调度升级为 per-widget schedule，为多个组件做准备。
