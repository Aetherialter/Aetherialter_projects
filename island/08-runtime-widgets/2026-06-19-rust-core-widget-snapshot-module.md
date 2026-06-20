# Rust core WidgetSnapshot 模块记录

## 结论

本次将临时写在 `lib.rs` 里的 UI 快照结构抽到 Rust `core` 模块：

```text
D:\Aether_island\src-tauri\src\core\mod.rs
D:\Aether_island\src-tauri\src\core\snapshot.rs
```

这一步让 `WidgetSnapshot` 成为后续 Runtime / Widget / Provider 都能复用的核心契约。

## 修改范围

```text
D:\Aether_island\src-tauri\src\lib.rs
D:\Aether_island\src-tauri\src\core\mod.rs
D:\Aether_island\src-tauri\src\core\snapshot.rs
```

## 新模块结构

```rust
pub mod snapshot;
```

```rust
use serde::Serialize;

#[derive(Clone, Copy, Debug, Eq, PartialEq, Serialize)]
#[serde(rename_all = "lowercase")]
pub enum WidgetTone {
    Ready,
}

#[derive(Clone, Debug, Eq, PartialEq, Serialize)]
pub struct WidgetSnapshot {
    pub title: String,
    pub subtitle: String,
    pub tone: WidgetTone,
}
```

## 为什么先只保留 Ready

前端已经支持：

```ts
type IslandTone = "ready" | "busy" | "warning" | "error";
```

但 Rust 当前只实际发送：

```rust
WidgetTone::Ready
```

如果 Rust enum 提前写入 `Busy` / `Warning` / `Error`，但当前没有构造这些 variant，`cargo clippy -- -D warnings` 会触发 dead code 错误。

因此当前 Rust 侧只保留实际使用的 `Ready`，等 Runtime 或 Widget 真正需要时再扩展。

## 所有权说明

`WidgetSnapshot` 拥有自己的 `String`：

```rust
pub title: String,
pub subtitle: String,
```

这样它可以安全跨线程移动到 Tauri event 发送逻辑里，不依赖临时字符串引用。

`WidgetTone` 是小型 enum，实现 `Copy`，传递时不需要堆分配。

## lib.rs 现在的职责

`lib.rs` 不再定义数据契约，只负责：

- 创建 Tauri app
- 注册插件和命令
- 在 setup 阶段发送 boot snapshot

数据结构由 `core::snapshot` 提供。

## 验证结果

以下命令已通过：

```powershell
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
pnpm build
cargo test --manifest-path ".\src-tauri\Cargo.toml"
```

## 下一步

可以继续新增：

```text
src-tauri/src/core/events.rs
src-tauri/src/core/widget.rs
```

逐步形成：

```text
WidgetSnapshot
WidgetEvent
IslandWidget trait
NativeWidgetProvider
```
