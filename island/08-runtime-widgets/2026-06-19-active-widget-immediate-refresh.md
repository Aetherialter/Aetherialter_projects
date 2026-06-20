# Active Widget Immediate Refresh

日期：2026-06-19

## 结论

本轮修正了 active widget 切换后的反馈链路：前端点击 widget 按钮后，Rust 不再只记录 active widget id，而是立即刷新 runtime 并向前端发出新的 `island://snapshot`。

这让交互从“按钮先高亮，内容等下一轮定时刷新”变成“按钮点击后立即切换内容”。

## Tauri 参数契约确认

当前前端调用：

```ts
await invoke("set_active_widget", { widgetId });
```

Rust 命令参数：

```rust
fn set_active_widget(widget_id: Option<String>, ...)
```

这是符合 Tauri 命令参数规则的。Tauri 官方文档说明，从前端传给 Rust command 的参数使用 camelCase；Rust 侧 snake_case 参数会映射到 JavaScript 侧 camelCase。

所以这里不是参数名 bug，`widgetId` 对应 `widget_id` 是正确的。

参考来源：

- Tauri v2 官方文档：`Calling Rust from the Frontend`
- Tauri v1 命令文档也明确记录过 snake_case 到 camelCase 的参数映射规则

## 改动内容

### Rust 命令层

`set_active_widget` 增加 `AppHandle` 注入，成功设置 active widget 后立即调用：

```rust
for event in runtime.refresh_events() {
    emit_frontend_event(&app_handle, event);
}
```

这表示：

1. 前端点击按钮。
2. Tauri command 进入 Rust。
3. Rust 更新 active widget。
4. Rust 立刻刷新 widgets。
5. Rust 立刻 emit snapshot 给前端。
6. 前端渲染新内容并同步高亮。

### Runtime 测试

新增测试：

```rust
refresh_events_returns_active_widget_snapshot_after_selection
```

它验证：

- `mock` widget 注册存在。
- 设置 active widget 为 `mock` 后。
- 下一次 `refresh_events()` 返回的 snapshot 是 `mock`。

Rust 单元测试数量从 11 个增加到 12 个。

## Rust 所有权解释

`RuntimeState` 里用：

```rust
runtime: Mutex<NativeRuntime>
```

意思是：运行时状态只有一个拥有者，外部通过互斥锁临时拿到可变访问权。

在 `set_active_widget` 命令里：

1. 先拿到 `MutexGuard<NativeRuntime>`。
2. 调用 `runtime.set_active_widget(widget_id)` 修改内部 active id。
3. 同一把锁内调用 `runtime.refresh_events()` 生成事件。
4. 把事件逐个 emit 给前端。

这里没有 clone 整个 runtime，也没有多个线程同时改 runtime 的问题。锁的代价很小，换来的是状态一致。

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
- Rust 单元测试 12 个全部通过。
- Clippy 以 `-D warnings` 通过。
- 前端 TypeScript 与 Vite production build 通过。

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

这一步让 widget switcher 真正成为可用交互，而不是只改变前端局部样式。

后续继续塞 GitHub、番茄钟、脚本进度等组件时，用户点击某个组件，内容会立即切到对应组件，这符合桌面工具的直觉。

## 下一步建议

下一步应做真实窗口验收：

```powershell
pnpm tauri dev
```

人工检查：

- 胶囊是否顶部居中。
- hover 是否展开。
- Clock / Mock / Sidecar 按钮是否可点击。
- 点击按钮后内容是否立即变化。
- 鼠标离开后是否正确折叠。
- 是否仍然无边框、无任务栏入口。
