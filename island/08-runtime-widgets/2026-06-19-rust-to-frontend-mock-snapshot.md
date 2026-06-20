# Rust 到前端 mock snapshot 链路记录

## 结论

本次打通了 Aether_island 第一条数据链路：

```text
Rust setup
→ emit island://snapshot
→ 前端 listen
→ 校验 payload
→ renderSnapshot
→ 更新胶囊 UI
```

这不是完整 Widget runtime，但它证明了后续 Runtime / Widget 可以通过同一条事件链路驱动前端。

## 修改范围

```text
D:\Aether_island\src-tauri\src\lib.rs
D:\Aether_island\src-tauri\capabilities\default.json
D:\Aether_island\src\main.ts
```

## Rust 侧实现

新增最小事件 payload：

```rust
#[derive(Clone, Serialize)]
struct IslandSnapshot {
    title: String,
    subtitle: String,
    tone: IslandTone,
}
```

当前 Rust 只构造实际使用的状态：

```rust
enum IslandTone {
    Ready,
}
```

这样可以通过 `cargo clippy -- -D warnings`，避免为未来预留未使用 enum variant 导致 dead code 错误。

## 事件发送

Rust 在 Tauri `setup` 中延迟 350ms 发送：

```rust
app_handle.emit("island://snapshot", IslandSnapshot::boot_ready());
```

延迟的原因是避免窗口和前端监听器尚未准备好时过早发送事件。

## 前端监听

前端监听：

```ts
listen<unknown>("island://snapshot", (event) => {
  if (!isIslandSnapshot(event.payload)) {
    console.warn("Ignored invalid island snapshot.", event.payload);
    return;
  }

  renderSnapshot(event.payload);
});
```

前端保留完整 tone 契约：

```ts
type IslandTone = "ready" | "busy" | "warning" | "error";
```

原因是前端 UI 已经支持这些显示状态；Rust 后续真实用到时再逐步扩展。

## 权限配置

`default.json` 显式加入：

```json
"core:event:allow-listen"
```

虽然 `core:default` 已包含 event 默认权限，但显式声明能让事件链路的意图更清楚。

## 验证结果

以下命令已通过：

```powershell
pnpm build
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
```

## 人工验收项

仍需运行：

```powershell
pnpm tauri dev
```

检查启动后副标题是否从：

```text
standby
```

变成：

```text
shell ready
```

## 下一步

下一阶段可以把 Rust 侧 `IslandSnapshot` 移入独立 `core` 模块，并扩展为真正的 `WidgetSnapshot`。
