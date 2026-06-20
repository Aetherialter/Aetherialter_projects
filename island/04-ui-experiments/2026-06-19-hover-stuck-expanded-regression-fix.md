# Hover Stuck Expanded Regression Fix

日期：2026-06-19

## 问题

用户反馈：测试几次后灵动岛会卡在展开状态，变成严重 bug。

这是上一轮透明边界修复引入的回归。

## 根因

上一轮逻辑是：

1. `pointerleave` 后延迟调用 `collapseIsland()`。
2. `collapseIsland()` 会检查鼠标是否还在外层安全区。
3. 如果还在安全区，就取消 collapse。

问题在于：

- 如果鼠标随后离开了窗口或透明区域，可能不会再触发岛上的 `pointermove`。
- 已经取消的 collapse 不会自动重新检查。
- 于是岛保持 hovered 状态，卡住不收起。

## 修改

文件：

```text
D:\Aether_island\src\main.ts
```

### 1. collapse 计时器可重入重置

`collapseIsland()` 现在会先清掉旧 timer，避免多个 timer 交错。

### 2. 安全区内持续重试

如果延迟检查时鼠标仍在安全区，不再永久取消，而是重新调度下一次 collapse 检查。

### 3. 全局 pointermove 追踪

新增 `document.addEventListener("pointermove", ...)`。

当岛处于 hovered 状态时，只要鼠标移动到安全区外，就会调度 collapse。这样即使鼠标已经不在岛元素上，也能继续恢复状态。

### 4. elementFromPoint 辅助判断

`shouldKeepExpandedFromPointer()` 现在先用：

```ts
document.elementFromPoint(lastPointerX, lastPointerY)
```

如果鼠标当前位置下的元素仍属于 island，就保持展开；否则再用安全矩形判断。

## 小白解释

上一版像是：系统问了一次“鼠标还在附近吗？”如果答案是“在”，它就不再问了。

这次改成：如果鼠标还在附近，就过一会儿继续问；只要鼠标真的走远，就收起来。

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
- 前端 build 通过。
- Python 7 个测试通过。
- 第一次 `pnpm smoke:window` 被已有当前项目 debug 实例阻止。
- 确认进程路径为 `D:\Aether_island\src-tauri\target\debug\aether-island.exe` 后关闭。
- 第二次 `pnpm smoke:window` 通过。
- 主窗口 `Layered=true`、`TopMost=true`、`ToolWindow=true` 等断言通过。

## 剩余风险

这个修复解决的是“展开后不收起”的状态机 bug。hover 手感仍需要用户真实鼠标测试确认。如果仍然觉得边界敏感，下一步应降低 `hoveredSafePadding` 或将 expanded/collapsed 的触发区域分成两套更明确的几何规则。
