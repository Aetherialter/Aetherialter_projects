# WidgetSnapshot 优先级选择策略记录

## 结论

本次给 widget snapshot 增加了：

```text
widget_id
priority
```

并在 provider 层实现了“刷新多个 widget，但只向前端发送最高优先级 snapshot”的策略。

这解决了多个 widget 同时刷新时互相覆盖 UI 的问题。

## 修改范围

```text
D:\Aether_island\src-tauri\src\core\snapshot.rs
D:\Aether_island\src-tauri\src\core\widget.rs
D:\Aether_island\src-tauri\src\core\provider.rs
D:\Aether_island\src\main.ts
```

## Snapshot 契约变化

旧结构：

```rust
WidgetSnapshot {
    title,
    subtitle,
    tone,
}
```

新结构：

```rust
WidgetSnapshot {
    widget_id,
    title,
    subtitle,
    tone,
    priority,
}
```

### widget_id

用于标识 snapshot 来源。

例如：

```text
system-clock
mock
```

后续接 GitHub、Pomodoro、Clipboard 时，这个字段能帮助 runtime 和前端知道状态来自哪个 widget。

### priority

用于决定多个 snapshot 同时产生时，谁应该显示在岛上。

当前约定：

```text
SystemClockWidget priority = 50
MockWidget priority = 10
```

因此时钟会优先显示，mock 只作为低优先级调试心跳。

## Provider 选择策略

`NativeWidgetProvider::refresh_all()` 现在仍会刷新所有 widget。

但是成功 snapshot 不会全部发送，而是：

```text
选择 priority 最大的 snapshot
```

错误事件仍然保留并上报。

简化理解：

```text
刷新所有组件
  -> 收集错误
  -> 成功结果里挑 priority 最大的
  -> 发给前端
```

## 为什么放在 Provider 层

Provider 是 widget 管理者，天然知道当前有哪些 widget。

如果把选择策略放到前端，前端就要理解所有 widget 的优先级和业务含义，会让 UI 层变重。

如果放在每个 widget 内部，单个 widget 又不知道其他 widget 的状态。

所以当前选择策略放在 provider 层最合适。

## 前端契约兼容

前端 `IslandSnapshot` 增加可选字段：

```ts
widget_id?: string;
priority?: number;
```

当前 UI 仍然只渲染：

```text
title
subtitle
tone
```

也就是说，前端协议向后扩展了，但视觉还保持不变。

## 当前限制

当前策略是非常简单的静态优先级。

它还没有考虑：

- 用户当前 focus 的 widget。
- hover 展开时是否展示更多信息。
- 某些错误状态是否应该临时抢占显示。
- Pomodoro 进行中是否应该高于时钟。
- GitHub 拉取完成后是否短暂弹出。

后续可以把 priority 扩展成动态评分：

```text
base_priority
freshness
urgency
user_focus
error_level
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

下一步可以新增：

```text
active_widget_id
```

让前端 hover 或用户选择时可以指定当前关注的 widget。

这样展开态就能从“自动最高优先级”升级成“用户当前关注优先”。
