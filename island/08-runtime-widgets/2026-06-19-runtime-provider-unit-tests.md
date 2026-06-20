# Runtime Provider Unit Tests

日期：2026-06-19

## 结论

本轮给 Rust `NativeWidgetProvider` 和 `NativeRuntime` 补了核心单元测试。v0.4 的重点不是先做更多花哨组件，而是让后续“小组件不断塞进来”时，调度规则不会变得不可预测。

测试覆盖了 active widget 选择、优先级 fallback、错误事件保留、shell 事件广播、shutdown 错误上报，以及 runtime 对未知 widget id 的拒绝。

## 为什么先测 Provider

`NativeWidgetProvider` 是当前 Rust-first 架构里的小组件调度中心。它决定：

- 哪些 widget 注册进运行时。
- 每轮刷新时选哪个 snapshot 给前端。
- active widget 是否覆盖默认优先级。
- widget 失败时是否保留错误信息。
- shell hover/collapse/shutdown 是否广播到所有 widget。

如果这里没有测试，后续新增 GitHub、番茄钟、剪贴板、脚本进度条等组件时，很容易出现“单个组件能跑，但整体选择逻辑坏掉”的问题。

## Rust 所有权解释

当前 provider 持有：

```rust
Vec<Box<dyn IslandWidget>>
```

用小白术语讲：

- `Vec` 是一个列表。
- `Box` 表示 widget 被放在堆上，由 provider 拥有。
- `dyn IslandWidget` 表示这些 widget 具体类型可以不同，只要都遵守 `IslandWidget` 接口。

所以 provider 不关心里面是 `SystemClockWidget`、`MockWidget`，还是未来的 `GitHubWidget`。它只通过 trait 方法调用：

- `id()`
- `title()`
- `priority()`
- `refresh()`
- `handle_shell_event()`
- `shutdown()`

测试里的 `TestWidget` 也是同样方式塞进 provider。这样测试不需要打开真实 Tauri 窗口，也不需要触发 Win32 API，就能验证调度规则。

## 新增测试点

### 1. 无 active widget 时选择最高优先级

当多个 widget 都刷新成功时，provider 选择 `priority` 最大的 snapshot。

这个规则让默认展示有稳定顺序，例如系统时钟可以优先于低优先级 sidecar 状态。

### 2. active widget 优先于最高优先级

当用户在 hover 展开区域选择某个 widget 时，即使它优先级较低，也应该展示它。

这保证了用户交互优先于系统默认选择。

### 3. 失败 widget 不阻断 fallback

如果某个 widget 刷新失败，provider 会产生 `WidgetFailed` 事件，同时继续选择可用 snapshot。

这对后续插件很重要：GitHub API 挂了，不应该让整个岛空白。

### 4. shell 事件广播保留错误

hover、collapse、shutdown 事件会广播到所有 widget。测试确认广播事件本身先进入事件流，widget 处理失败也会被记录。

### 5. shutdown 错误上报

如果某个 widget shutdown 失败，provider 会把错误转成 `WidgetFailed`。后续接 Python sidecar 时，这个点会很关键，因为子进程退出失败需要可观察。

### 6. runtime 拒绝未知 active widget

`NativeRuntime::set_active_widget` 会拒绝不存在的 widget id。

这能防止前端传入错误按钮值后污染 runtime 状态。

## 验证结果

在 `D:\Aether_island` 运行：

```powershell
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo test --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
pnpm build
```

结果：

- Rust 格式检查通过。
- Rust 单元测试从 0 个增加到 9 个，全部通过。
- Clippy 以 `-D warnings` 通过。
- 前端 production build 通过。

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

## 对 v0.4 的意义

这一步让 v0.4 的“可扩展小组件框架”更接近可维护状态：

1. 小组件接口不是只停留在代码结构上，而是有行为测试保护。
2. 用户手动选择 widget 的行为被明确固定。
3. 单个 widget 失败不会拖垮整体展示。
4. 未来接 Python sidecar 时，shutdown 和错误上报已经有测试方向。

## 下一步建议

下一步可以做两个方向之一：

1. 补 `FrontendSnapshot` 的完整字段透传，让前端拿到 `widget_id`、`priority`、`tone`，为更复杂 UI 状态做准备。
2. 做一次 `pnpm tauri dev` 的人工交互验证，确认动态 widget 切换按钮在真实窗口中符合预期。
