# Aether_island 隐藏内部 Widget 入口

## 结论

本次将 `mock` 和 `sidecar` 从前端 widget 切换列表中隐藏。

注意：这不是删除能力。`mock` 和 `sidecar` 仍保留在 runtime 内部，后续开发、测试和 Python sidecar 接入还可以继续使用。

## 实现方式

`IslandWidget` trait 新增：

```rust
fn is_visible(&self) -> bool {
    true
}
```

默认 widget 可见。

`mock` 和 `sidecar` 覆盖为：

```rust
fn is_visible(&self) -> bool {
    false
}
```

`NativeWidgetProvider::widget_descriptors()` 只返回可见 widget：

```rust
.filter(|widget| widget.is_visible())
```

## 当前 UI 可见 Widget

```text
Clock
GitHub
```

## 当前内部保留 Widget

```text
Mock
Sidecar
```

## 防屎山措施

- 没有在前端写死 `id !== "mock"` 这种过滤逻辑。
- 可见性属于 widget 元数据，由 Rust runtime 统一暴露。
- 前端只消费 descriptor，不关心哪些 widget 是内部保留项。
- 后续可以把 `is_visible()` 升级为配置项，而不用改前端逻辑。

## 验证结果

已通过：

```powershell
pnpm check
```

结果：

- Rust tests：24 passed。
- Rust clippy：passed。
- Frontend build：passed。
- Python lint/tests：passed。
