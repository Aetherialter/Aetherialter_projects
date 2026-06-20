# Frontend Snapshot Contract Hardening

日期：2026-06-19

## 结论

本轮把 Rust 到前端的 snapshot 契约从“只够显示文字”升级为“完整组件状态包”。前端现在可以稳定拿到：

- `widget_id`
- `title`
- `subtitle`
- `tone`
- `priority`

这一步对 v0.4 很关键，因为后续继续塞小组件时，前端不应该靠猜测或局部状态判断当前展示的是哪个 widget。

## 改动内容

### Rust 侧

`FrontendSnapshot` 新增并透传：

```rust
pub struct FrontendSnapshot {
    pub widget_id: &'static str,
    pub title: String,
    pub subtitle: String,
    pub tone: WidgetTone,
    pub priority: u8,
}
```

`RuntimeEvent::SnapshotUpdated` 现在保留来自 `WidgetSnapshot` 的完整字段：

- widget id 不丢失。
- tone 不再硬编码成 `"ready"`。
- priority 会传给前端。

`RuntimeEvent::WidgetFailed` 会被转换成一个明确的错误 snapshot：

- `widget_id = "runtime-error"`
- `tone = WidgetTone::Error`
- `priority = 0`

这样前端可以显示错误状态，但不会把错误状态误当成某个正常 widget 的 active 状态。

### 前端侧

`IslandSnapshot` 从可选字段升级为必需字段：

```ts
type IslandSnapshot = {
  widget_id: string;
  title: string;
  subtitle: string;
  tone: IslandTone;
  priority: number;
};
```

前端校验函数也同步收紧：缺少 `widget_id` 或 `priority` 的事件会被拒绝。

渲染时，如果收到的不是 `runtime-error`，就用 `widget_id` 自动高亮当前 widget 按钮。

## 小白解释

之前的数据像这样：

```text
显示：标题、描述、颜色
```

能看，但不知道“这是谁发来的”。

现在的数据像这样：

```text
显示：哪个组件、标题、描述、颜色、优先级
```

这就像每个快递包裹都有寄件人和编号。后面小组件多了以后，前端才能准确知道当前展示的是时钟、mock、sidecar，还是未来的 GitHub / 番茄钟 / 脚本进度。

## Rust 所有权与类型点

`RuntimeEvent::SnapshotUpdated(snapshot)` 这里会取得 `WidgetSnapshot` 的所有权。

因为 `snapshot.title` 和 `snapshot.subtitle` 是 `String`，它们不是复制类型。转换成 `FrontendSnapshot` 时直接移动进去，避免多余 clone。

`widget_id` 是 `&'static str`，表示它是程序生命周期内稳定存在的字符串字面量或静态字符串。当前 widget id 都是固定 id，例如：

- `system-clock`
- `mock`
- `sidecar`

所以这里可以安全透传给前端序列化。

## 新增测试

Rust runtime 新增 2 个测试：

1. `runtime_event_to_frontend_preserves_snapshot_contract_fields`
   - 验证正常 snapshot 转前端事件时不会丢失 `widget_id`、`tone`、`priority`。

2. `runtime_event_to_frontend_marks_widget_failures_as_error_snapshots`
   - 验证 widget 失败会变成明确的 error snapshot。

Rust 单元测试数量从 9 个增加到 11 个。

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
- Rust 单元测试 11 个全部通过。
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

这一步把 Rust Runtime 和前端之间的公共契约进一步固定下来：

1. 前端能知道当前展示的 widget 身份。
2. 错误状态和正常 widget 状态不会混淆。
3. 后续新增 widget 时不用改前端 snapshot 基础类型。
4. runtime 到前端的数据转换有单元测试保护。

## 下一步建议

下一步可以做真实窗口人工验证：运行 `pnpm tauri dev`，检查 hover 展开、widget 切换按钮高亮、时钟刷新、错误状态样式是否符合预期。这个属于 UI 行为验收，单靠 build 不能完全证明。
