# 动态 Widget 列表下发记录

## 结论

本次把前端切换器从硬编码：

```text
Clock
Mock
```

升级为从 Rust runtime 动态获取 widget 列表。

现在新增 widget 后，只要它注册到 provider，前端切换器就可以自动渲染出来。

## 修改范围

```text
D:\Aether_island\src-tauri\src\core\widget.rs
D:\Aether_island\src-tauri\src\core\provider.rs
D:\Aether_island\src-tauri\src\core\runtime.rs
D:\Aether_island\src-tauri\src\lib.rs
D:\Aether_island\index.html
D:\Aether_island\src\main.ts
```

## Rust 新增 WidgetDescriptor

新增：

```rust
WidgetDescriptor {
    id,
    title,
    priority,
}
```

它描述一个 widget 的前端展示元数据。

## IslandWidget trait 变化

新增：

```rust
fn priority(&self) -> u8;
```

这样 provider 不需要等 widget 刷新后才知道优先级。

当前：

```text
SystemClockWidget priority = 50
MockWidget priority = 10
```

## Provider / Runtime 变化

Provider 新增：

```rust
widget_descriptors()
```

Runtime 新增：

```rust
widget_descriptors()
```

Tauri command 新增：

```rust
list_widgets()
```

返回：

```json
[
  { "id": "system-clock", "title": "System Clock", "priority": 50 },
  { "id": "mock", "title": "Mock", "priority": 10 }
]
```

## 前端变化

HTML 中的按钮不再硬编码，只保留容器：

```html
<div id="widget-switcher" class="island__switcher"></div>
```

TypeScript 启动后调用：

```ts
invoke("list_widgets")
```

然后动态创建按钮。

## 为什么这一步重要

这一步把前端从“知道有哪些 widget”里解耦出来。

后续新增：

```text
GitHubWidget
PomodoroWidget
ClipboardWidget
PythonSidecarWidget
```

只需要在 Rust provider 注册，前端就能自动出现对应切换项。

这是插件化/组件化 UI 的基础。

## 当前限制

当前 `WidgetDescriptor` 还很简单，只包含：

```text
id
title
priority
```

后续可能需要增加：

```text
short_label
icon
category
enabled
supports_expanded_view
```

但现在先保持轻量。

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

检查：

- hover 后切换器是否仍显示 Clock / Mock。
- 点击切换是否仍有效。
- 后续新增 widget 后是否能自动出现在切换器里。

## 下一步建议

下一步适合开始整理 v0.4 的“sidecar-ready”边界。

可以新增一个空的：

```text
SidecarWidget
```

先不启动 Python，只定义 Rust 侧生命周期和 IPC 数据结构，为后续 Python 双进程做准备。
