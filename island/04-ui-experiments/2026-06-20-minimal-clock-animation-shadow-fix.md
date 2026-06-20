# Aether_island 最小 Clock 动画与阴影修正

## 问题

最小 Clock 基线收束后，动画反馈变弱，同时岛左右下角仍能看到外投影。

原因：

- hover 状态仍存在，但 CSS 没有给 `.island--hovered` 明确视觉反馈。
- `.island__surface` 保留了外部 `box-shadow`，透明窗口中会在下角显得像残留阴影。

## 本次修正

### 1. 去除外投影

删除外部阴影：

```css
0 10px 24px rgba(0, 0, 0, 0.2)
```

改为只保留内部高光和内阴影。

### 2. 恢复轻量 hover 动画

hover 时只在 `292 x 44` 岛范围内做视觉反馈：

- 背景略微变亮。
- 轻微 `scaleX/scaleY` 形变。
- 内容上移 1px。
- 时间文字亮度提升。

不扩大真实窗口，不增加交互区域。

## 当前硬约束

```text
视觉范围 = 292 x 44
窗口范围 = 292 x 44
交互范围 = 292 x 44
```

动画只能发生在岛内部，不能靠扩大透明窗口实现。

## 验证结果

已通过：

```powershell
pnpm check
pnpm smoke:window
```

烟测确认：

```text
Width=292
Height=44
```
