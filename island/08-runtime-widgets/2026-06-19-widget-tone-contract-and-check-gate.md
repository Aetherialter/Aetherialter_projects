# Widget Tone Contract and Check Gate Fix

日期：2026-06-19

## 结论

本轮把 Aether_island 的组件状态颜色契约收束成四个稳定值：

```text
ready | busy | warning | error
```

这四个值现在同时被 Rust runtime、TypeScript 前端和 Python sidecar 草稿使用。这样后续继续添加小组件时，不需要每个组件各自发明状态字段，也不会出现 Rust 发一种状态、前端只认识另一种状态的问题。

同时修复了 `pnpm check` 的一个质量门禁问题：PowerShell 默认不会把所有原生命令的非零退出码自动转成异常，所以脚本现在会显式检查 `$LASTEXITCODE`，确保 `cargo`、`pnpm`、`uv` 任一步失败时整个门禁真正失败。

## 改动范围

### Rust runtime

文件：

```text
D:\Aether_island\src-tauri\src\core\snapshot.rs
D:\Aether_island\src-tauri\src\core\widget.rs
```

`WidgetTone` 扩展为：

```rust
pub enum WidgetTone {
    Ready,
    Busy,
    Warning,
    Error,
}
```

`WidgetSnapshot::with_tone(...)` 用于需要显式指定状态的组件。

当前 `SidecarWidget` 的状态语义：

- `Ready(_)` -> `busy`
- `Stopped` -> `warning`

小白解释：

- `ready`：组件正常，展示的是稳定信息。
- `busy`：组件活着，但还在等外部数据或后台同步。
- `warning`：组件还能显示，但状态不理想，比如 sidecar 已停止。
- `error`：组件或 runtime 发生明确错误。

### Python sidecar

文件：

```text
D:\Aether_island\island-daemon\src\aether_island_daemon\protocol.py
D:\Aether_island\island-daemon\src\aether_island_daemon\main.py
```

`render_update(...)` 新增 `tone` 参数，默认值是 `ready`。

Python 输出的 `RENDER_UPDATE` payload 现在包含：

```json
{
  "plugin": "sidecar",
  "title": "Sidecar",
  "subtitle": "shell hovered",
  "tone": "ready",
  "priority": 1
}
```

这与前端 `IslandSnapshot` 的状态枚举保持一致。

### 质量门禁

文件：

```text
D:\Aether_island\scripts\check.ps1
```

`Invoke-Step` 现在会在每一步之后检查：

```powershell
if ($LASTEXITCODE -ne 0) {
    throw "$Name failed with exit code $LASTEXITCODE"
}
```

这次修改前曾出现过一个真实现象：Rust clippy 报错，但脚本继续执行后仍打印 `All checks passed.`。修复后，Rust 编译失败会让 `pnpm check` 直接失败，后续验证也证明该行为生效。

## Rust 知识点

### 1. 枚举不是字符串

Rust 里的 `WidgetTone::Ready`、`WidgetTone::Busy` 不是普通字符串，而是强类型枚举。

好处是：

- 写错状态名会在编译期暴露。
- match 分支可以被编译器检查。
- 对外序列化时再通过 serde 变成前端需要的字符串。

这里使用：

```rust
#[serde(rename_all = "lowercase")]
```

表示 Rust 里的 `WidgetTone::Warning` 序列化到 JSON 时会变成：

```json
"warning"
```

### 2. 所有权没有变复杂

`WidgetSnapshot` 拥有自己的 `String` 字段：

```rust
title: String,
subtitle: String,
```

构造 snapshot 时，标题和副标题会被移动进结构体。后续 runtime 发给前端时，serde 只是读取这些字段进行 JSON 序列化，不会改变所有权模型。

`widget_id` 仍然是 `&'static str`，适合当前固定 widget id：

```text
system-clock
mock
sidecar
```

这表示这些 id 在程序生命周期内稳定存在。

### 3. 测试中的构造不等于生产代码使用

这次第一次运行 `pnpm check` 时，Rust 单元测试能构造 `Busy` 和 `Warning`，但 clippy 仍然报生产代码中这些变体没有被使用。

这提醒我们：

- 测试证明契约能序列化。
- 生产代码使用证明契约不是死接口。

所以最终让 `SidecarWidget` 真实返回 `busy/warning`，而不是用 `#[allow(dead_code)]` 压掉警告。

## 验证结果

在 `D:\Aether_island` 运行：

```powershell
pnpm check
```

结果：

- Rust format 通过。
- Rust tests：18 个测试全部通过。
- Rust clippy：通过。
- Frontend build：通过。
- Python ruff format：通过。
- Python ruff check：通过。
- Python pytest：7 个测试全部通过。

随后运行：

```powershell
pnpm smoke:window
```

结果：

- 自动启动 Tauri dev 窗口。
- 等待 Win32 样式应用完成。
- `Aether_island window assertions passed.`
- `Window smoke passed.`

## 对 v0.4 的意义

这一步不是视觉大改，而是把后续小组件系统需要依赖的基础协议钉住：

1. Rust widget 能用统一 tone 描述状态。
2. Python sidecar 输出能对齐前端状态字段。
3. TypeScript 前端已经认识这四种状态。
4. 本地门禁现在更可信，不会在底层命令失败后误报成功。

## 下一步建议

v0.4 下一步适合继续做“预发布审查”：把当前明确支持和暂不支持的能力列成验收表，避免把最初 IslandGate 大设想和当前 Rust-first v0.4 混在一起。
