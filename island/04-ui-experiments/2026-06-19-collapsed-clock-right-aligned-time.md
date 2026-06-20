# Collapsed Clock Right-aligned Time

日期：2026-06-19

## 目标

用户希望常态岛的 Clock 展示更像状态条，而不是透明小标签：

- 左侧显示 `Clock`。
- 右侧显示当前时间并右对齐。
- 去掉 Rust 输出里的 `local` 前缀。
- 时间字号更大、更清晰，减少过强的透明感。

## 修改

### Rust Clock 输出

文件：

```text
D:\Aether_island\src-tauri\src\core\widget.rs
```

将系统时钟 subtitle 从：

```text
local HH:MM:SS
```

改为：

```text
HH:MM:SS
```

这样前端常态显示不再有多余说明文字。

### CSS 常态布局

文件：

```text
D:\Aether_island\src\styles.css
```

collapsed 状态下：

- `.island__content` 使用 `justify-content: space-between`。
- 标题留在左侧。
- subtitle 使用 `margin-left: auto` 和 `text-align: right` 靠右。
- subtitle 字号提升到 `14px`。
- 使用 `font-variant-numeric: tabular-nums`，让时间数字宽度稳定，刷新秒数时不抖动。
- 背景透明度从偏透明改为更实：`rgba(16, 17, 22, 0.92)`。
- 整体 opacity 从 `0.86` 提高到 `0.96`。

## 小白解释

之前常态像：

```text
Clock local 20:18:31
```

而且时间更像淡淡贴在上面。

现在目标是：

```text
Clock              20:18:31
```

也就是左边告诉你当前组件是谁，右边用更清楚的数字显示核心状态。

`tabular-nums` 可以理解成“每个数字占同样宽度”。如果不用它，`1` 可能比 `8` 窄，秒数跳动时文本会轻微左右晃。

## 验证

在 `D:\Aether_island` 运行：

```powershell
pnpm check
pnpm smoke:window
```

结果：

- 第一次 `pnpm check` 因 Rust 格式未满足 `rustfmt` 失败。
- 按 rustfmt 建议调整后，`pnpm check` 通过。
- `pnpm smoke:window` 第一次被已有当前项目 debug 实例阻止。
- 确认该实例路径为 `D:\Aether_island\src-tauri\target\debug\aether-island.exe` 后关闭。
- 第二次 `pnpm smoke:window` 通过。

窗口探针仍验证主窗口：

- `ToolWindow=true`
- `TopMost=true`
- `Layered=true`
- `Transparent=false`
- `AppWindow=false`
- `NoActivate=true`
- `Width=480`
- `Height=120`

## 剩余风险

本轮验证证明构建、测试和窗口样式正常；视觉细节仍建议在真实 `pnpm tauri dev` 窗口中人工看一眼，尤其是不同 DPI 缩放下的右对齐间距。
