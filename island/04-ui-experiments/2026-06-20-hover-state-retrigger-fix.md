# Aether_island Hover 动画只能触发一次的修复

## 问题

最小 Clock 基线中，窗口范围已经严格等于岛范围：

```text
292 x 44
```

旧的 `HoverController` 仍然依赖：

- `document.pointermove`
- 窗口外安全区判断
- `elementFromPoint`

但鼠标离开这个小窗口后，页面不一定能继续收到窗口外的 pointermove。因此 hovered 状态可能卡住，导致动画只在第一次进入时触发，后续状态没有回到 collapsed，自然不会再次播放。

## 修复

将 hover 状态机简化为最小规则：

```text
pointerenter -> hovered
pointerleave -> 延迟 collapsed
```

删除：

- 外部 pointermove 追踪。
- surface 命中判断。
- collapsed/hovered safe padding。
- elementFromPoint 判断。

## 当前交互语义

```text
鼠标进入岛范围：播放 hover 动画
鼠标离开岛范围：180ms 后收起
再次进入：再次播放 hover 动画
```

这和当前硬约束一致：

```text
交互范围 = 岛范围 = 窗口范围
```

## 验证结果

已通过：

```powershell
pnpm check
pnpm smoke:window
```

窗口仍为：

```text
292 x 44
```
