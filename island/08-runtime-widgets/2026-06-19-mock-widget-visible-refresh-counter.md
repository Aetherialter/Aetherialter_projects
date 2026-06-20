# MockWidget 可见刷新计数记录

## 结论

本次让 `MockWidget` 在每次刷新时显示递增计数。

之前 mock widget 每次都返回：

```text
shell ready
```

即使后台周期刷新线程正常工作，前端看起来也没有变化。

现在它会返回：

```text
refresh #1
refresh #2
refresh #3
```

这样运行 `pnpm tauri dev` 时，可以肉眼确认周期刷新链路是否真的在工作。

## 修改范围

```text
D:\Aether_island\src-tauri\src\core\snapshot.rs
D:\Aether_island\src-tauri\src\core\widget.rs
```

## Snapshot 变化

`WidgetSnapshot` 新增通用构造函数：

```rust
WidgetSnapshot::ready(subtitle)
```

它统一生成 ready 状态的 snapshot。

旧的：

```rust
boot_ready()
```

因为不再使用，已删除。这样可以通过：

```powershell
cargo clippy -- -D warnings
```

避免死代码。

## MockWidget 变化

`MockWidget` 从空结构体变成有内部状态：

```rust
pub struct MockWidget {
    refresh_count: u64,
}
```

每次：

```rust
refresh()
```

都会：

```text
refresh_count += 1
```

并把计数写入 subtitle。

## Rust 所有权解释

`refresh()` 的签名是：

```rust
fn refresh(&mut self) -> WidgetResult<WidgetSnapshot>
```

这里必须是 `&mut self`，因为刷新会修改内部计数器。

小白理解：

- `&self`：只看，不改。
- `&mut self`：我要改自己的内部状态。

`MockWidget` 被放在：

```rust
Box<dyn IslandWidget>
```

里面，由 provider 持有。provider 每次刷新时通过 `iter_mut()` 拿到可变借用，所以计数能持续累加。

## 对调试的意义

这是一个可观察性改动，不是最终业务功能。

它的作用是验证整条链路：

```text
后台刷新线程
  -> NativeRuntime::refresh_events()
  -> MockWidget::refresh()
  -> WidgetSnapshot
  -> island://snapshot
  -> TypeScript renderSnapshot
  -> UI subtitle 变化
```

如果 UI 每 5 秒看到一次数字增长，说明周期刷新链路真实可用。

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

观察胶囊副标题是否从：

```text
standby
```

变成：

```text
refresh #1
refresh #2
...
```

当前 mock widget 的刷新间隔是 5 秒。

## 下一步建议

下一步可以把 mock widget 拆成更接近真实组件的示例：

```text
SystemClockWidget
```

它不需要网络和密钥，能稳定显示当前时间，适合作为 v0.4 前的第一个真实本地 widget。
