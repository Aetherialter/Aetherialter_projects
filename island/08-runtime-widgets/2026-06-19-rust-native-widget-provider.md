# Rust NativeWidgetProvider 运行时入口记录

## 结论

本次新增 `NativeWidgetProvider`，把原来散落在 `lib.rs` 里的 mock widget 操作收进 Rust core 层。

这一步的意义不是增加视觉功能，而是把 Aether_island 从“单个 mock 事件”推进到“有 Widget Runtime 入口”的结构。

## 修改范围

```text
D:\Aether_island\src-tauri\src\core\provider.rs
D:\Aether_island\src-tauri\src\core\mod.rs
D:\Aether_island\src-tauri\src\lib.rs
```

## 当前模块关系

```text
lib.rs
  -> NativeWidgetProvider
      -> Vec<Box<dyn IslandWidget>>
          -> MockWidget
      -> RuntimeEvent
          -> island://snapshot
              -> frontend renderSnapshot
```

## NativeWidgetProvider 职责

`NativeWidgetProvider` 现在负责：

- 注册原生 widget。
- 持有所有 widget 的所有权。
- 广播 shell 状态，例如 hovered、collapsed、shutdown requested。
- 刷新所有 widget，并把结果转成 `RuntimeEvent`。
- 统一关闭所有 widget。

## Rust 所有权解释

核心字段是：

```rust
widgets: Vec<Box<dyn IslandWidget>>
```

小白理解：

- `Vec` 是一个动态数组，用来放多个组件。
- `Box` 表示组件对象放在堆上，provider 拿着它的所有权。
- `dyn IslandWidget` 表示这里不关心具体类型，只要求它满足 `IslandWidget` 这套接口。

因此后续可以同时放：

```text
MockWidget
GithubWidget
PomodoroWidget
ClipboardWidget
```

只要它们都实现 `IslandWidget`，provider 就可以统一管理。

## 为什么 refresh_all 使用 iter_mut

刷新接口是：

```rust
fn refresh(&mut self) -> WidgetResult<WidgetSnapshot>;
```

`&mut self` 表示刷新时 widget 可能会修改自己的内部状态，例如：

- 上次刷新时间。
- 本地缓存。
- 倒计时剩余秒数。
- 网络错误次数。

provider 使用：

```rust
self.widgets.iter_mut()
```

含义是：provider 不把 widget 拿走，只是逐个临时借用为可修改状态。刷新结束后，组件仍然归 provider 管。

## 为什么还没有引入 async

当前 `MockWidget` 只是同步返回一条 boot snapshot。

如果现在引入 `async-trait`、tokio 调度器或复杂任务队列，会让结构提前变重。当前更合理的阶段是先稳定这些边界：

- `IslandWidget` trait。
- `WidgetSnapshot` 数据契约。
- `RuntimeEvent` 事件契约。
- `NativeWidgetProvider` 管理入口。

等接入 GitHub、端口监听、剪贴板 hook 等真实异步需求时，再决定是否使用 async runtime。

## 和 Python sidecar 的关系

当前 provider 是 Rust 原生 widget 管理器。

未来如果接 Python，可以新增一种 widget，例如：

```text
PythonSidecarWidget
```

它依然实现 `IslandWidget`，内部再负责：

- 启动 Python 进程。
- 读写 stdio JSON。
- 管理子进程生命周期。
- 把 Python 数据转换成 `WidgetSnapshot`。

这样前端和 provider 不需要知道 Python 的存在。

## 验证结果

以下命令已通过：

```powershell
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
pnpm build
```

## 当前剩余限制

- 当前 provider 只有 `MockWidget`，还没有真实业务 widget。
- 当前刷新是启动时单次触发，还没有周期调度。
- 当前 shell state 只在 Rust 内部流转，还没有从前端 hover 主动回传到 Rust。
- 当前还没有 Win32 鼠标穿透、托盘、自动启动。

## 下一步建议

下一步可以在 provider 基础上做两件事之一：

1. 加 `core/runtime.rs`，把启动、刷新、shutdown 编排从 `lib.rs` 再下沉一层。
2. 加前端到 Rust 的 shell state command，让 hover/collapse 不只是 CSS 状态，而能进入 Rust runtime。
