# Rust NativeRuntime 编排层记录

## 结论

本次新增 `NativeRuntime`，把启动阶段的 widget 编排逻辑从 `lib.rs` 下沉到 Rust core 层。

现在 `lib.rs` 更接近 Tauri 外壳入口，只负责：

- 创建 Tauri app。
- 在 setup 阶段启动后台线程。
- 把 runtime 产出的前端事件 emit 给 WebView。

真正的 widget 生命周期编排交给 `NativeRuntime`。

## 修改范围

```text
D:\Aether_island\src-tauri\src\core\runtime.rs
D:\Aether_island\src-tauri\src\core\mod.rs
D:\Aether_island\src-tauri\src\lib.rs
```

## 当前调用链

```text
Tauri setup
  -> emit_boot_snapshot
      -> NativeRuntime::new()
          -> NativeWidgetProvider::new()
              -> MockWidget
      -> NativeRuntime::boot_events()
      -> emit island://snapshot
      -> frontend listen
      -> renderSnapshot
```

## 模块职责边界

### lib.rs

`lib.rs` 是 Tauri 应用入口层。

它不应该长期承担：

- widget 注册。
- refresh 调度。
- shell 状态广播。
- Python sidecar 生命周期。
- 业务数据转换。

这些职责继续放在 `lib.rs` 会让入口文件越来越胖，后面接托盘、Win32、Python 时会变得难维护。

### NativeRuntime

`NativeRuntime` 是 Rust 原生运行时编排层。

当前负责：

- 持有 `NativeWidgetProvider`。
- 执行 boot 阶段事件流。
- 把内部 `RuntimeEvent` 转换成前端能消费的 `FrontendEvent`。

### NativeWidgetProvider

`NativeWidgetProvider` 是 widget 管理器。

它负责：

- 持有所有 widget。
- 调用 widget 的 refresh。
- 广播 shell event。
- 收集 widget 错误。

## 为什么新增 FrontendEvent

内部事件和前端事件不能完全混在一起。

内部事件：

```text
RuntimeEvent
```

表达 Rust runtime 内部发生了什么，例如：

- Widget 刷新成功。
- Widget 失败。
- Shell 状态变化。

前端事件：

```text
FrontendEvent
```

表达 WebView 应该看到什么。

目前只有：

```text
FrontendEvent::Snapshot
```

这样做的好处是，未来即使内部事件变复杂，前端协议也可以保持稳定。

## Rust 所有权解释

`NativeRuntime` 拥有：

```rust
provider: NativeWidgetProvider
```

`NativeWidgetProvider` 再拥有：

```rust
Vec<Box<dyn IslandWidget>>
```

也就是说所有 widget 的所有权链路是：

```text
NativeRuntime
  owns NativeWidgetProvider
    owns Vec
      owns Box<dyn IslandWidget>
```

启动线程里创建 `NativeRuntime` 后，runtime 跟随线程生命周期存在。当前 boot 事件跑完后，runtime 会被释放，里面的 widget 也一起释放。

## 为什么现在还不是完整 Runtime

当前 `NativeRuntime::boot_events()` 只跑一次：

- hovered
- collapsed
- refresh_all
- shutdown requested
- shutdown_all

这仍然是启动链路验证，不是长期运行调度器。

真正的 v0.4 runtime 还需要：

- 周期刷新。
- 前端 shell state 回传。
- 多 widget 注册。
- 本地缓存。
- 错误状态和降级显示。
- 可选 Python sidecar 生命周期管理。

## 为什么暂时不直接上 Python 双进程

Python sidecar 是后续扩展点，不是当前结构的前置条件。

当前先把 Rust runtime 边界做好，有两个好处：

1. 纯 Rust widget 可以直接接入，不被 Python 进程复杂度拖住。
2. 未来 Python 可以作为一种 `IslandWidget` 实现接入，而不是改变整个架构。

未来可能的结构：

```text
NativeWidgetProvider
  -> MockWidget
  -> GithubNativeWidget
  -> PythonSidecarWidget
       -> python daemon
       -> stdio json
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

下一步建议补前端到 Rust 的 shell state 通道：

```text
hover/collapse
  -> TypeScript invoke 或 emit
  -> Rust command
  -> NativeRuntime / provider
  -> widget handle_shell_event
```

这样 hover 就不只是 CSS 动画状态，而会变成 runtime 可感知的系统状态。
