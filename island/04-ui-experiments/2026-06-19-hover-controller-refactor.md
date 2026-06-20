# Hover Controller Refactor

日期：2026-06-19

## 背景

连续修 hover 抖动后，`src\main.ts` 里逐渐堆了多层逻辑：

- 当前 hover 状态。
- pointer 坐标。
- collapsed 可见区域判定。
- hovered 安全区域判定。
- collapse timer。
- `document.elementFromPoint`。
- `pointermove` / `pointerleave`。

这已经有“石山化”苗头。继续在 `main.ts` 里堆 `if` 会让后续调 hover 手感越来越危险。

## 修改

新增文件：

```text
D:\Aether_island\src\hover-controller.ts
```

把 hover 状态机抽成 `HoverController`：

```text
HoverController
├─ state: collapsed / hovered
├─ pointer position
├─ collapsedHitPadding
├─ hoveredSafePadding
├─ collapseDelayMs
├─ root pointermove / pointerleave
├─ document pointermove
└─ onStateChange 回调
```

`main.ts` 现在只负责：

- 渲染 snapshot。
- 绑定 widget switcher。
- 创建 `HoverController`。
- 在 `onStateChange` 中调用 `setIslandState`。

## 状态规则

### collapsed

只有鼠标进入当前可见 `.island__surface` 附近，才展开。

### hovered

鼠标仍在 island 元素或 expanded 安全区内，保持展开。

鼠标离开安全区超过 `collapseDelayMs`，收起。

如果检查时鼠标仍在安全区，不永久取消，而是继续调度下一次检查。

## 小白解释

之前主文件像把“显示内容”和“鼠标边界判断”揉在一起。

现在拆成：

```text
main.ts              负责业务显示
hover-controller.ts  负责鼠标状态机
```

以后如果要调手感，只看 `HoverController` 就行，不会把 widget、IPC、时间显示一起搅乱。

## 验证

在 `D:\Aether_island` 运行：

```powershell
pnpm check
pnpm smoke:window
```

结果：

- `pnpm check` 通过。
- Rust 18 个测试通过。
- Clippy 通过。
- 前端 build 通过，Vite 显示 11 个模块参与构建。
- Python 7 个测试通过。
- `pnpm smoke:window` 通过。
- 主窗口断言通过：`ToolWindow=true`、`TopMost=true`、`Layered=true`、`Transparent=false`、`AppWindow=false`、`NoActivate=true`。

## 剩余风险

这次是结构优化，不保证 hover 手感最终完美。它的价值是把后续调参点集中起来：

```text
collapsedHitPadding
hoveredSafePadding
collapseDelayMs
```

如果用户继续反馈边界敏感，可以只调整这三个参数或 `HoverController` 的几何规则。
