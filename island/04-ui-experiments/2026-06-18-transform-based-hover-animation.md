# Aether_island transform 驱动 hover 动画实现记录

## 结论

本次将 hover 动画从直接修改 `width` / `height`，调整为主要使用 `transform` / `opacity` 驱动。

核心目的：

```text
减少布局重排
让动画更多走合成层
降低 hover 反复触发时的抖动
```

## 修改范围

```text
D:\Aether_island\src\main.ts
D:\Aether_island\src\styles.css
```

## TypeScript 改动

### 1. 使用 requestAnimationFrame 合并 class 更新

现在状态切换不是立刻直接改 class，而是放到下一帧：

```ts
cancelAnimationFrame(pendingFrame);
pendingFrame = requestAnimationFrame(() => {
  island.classList.toggle("island--collapsed", state === "collapsed");
  island.classList.toggle("island--hovered", state === "hovered");
});
```

这样做的意义不是强行提高帧率，而是把状态变化集中到浏览器的渲染节奏里。

### 2. mouseleave 增加 80ms 延迟

鼠标离开时不会立刻收起，而是稍微等待：

```ts
collapseTimer = window.setTimeout(() => {
  setIslandState("collapsed");
}, 80);
```

这样可以减少鼠标在边缘来回移动时的视觉抖动。

## CSS 改动

### 1. 固定真实布局尺寸

`.island` 的真实布局尺寸固定为展开尺寸：

```css
width: 220px;
height: 44px;
```

折叠态不再靠改宽高，而是靠缩放：

```css
transform: scaleX(0.73) scaleY(0.73);
```

hover 态恢复：

```css
transform: scale(1);
```

### 2. 优先动画 transform / opacity

现在 transition 只覆盖关键属性：

```css
transition:
  transform 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275),
  opacity 0.26s ease,
  background-color 0.26s ease,
  box-shadow 0.26s ease;
```

不再使用 `transition: all`，避免浏览器尝试动画不必要的属性。

### 3. 降低 blur 和阴影成本

`backdrop-filter` 从 `blur(18px)` 降低到 `blur(12px)`。

阴影也变轻，减少透明窗口合成压力。

### 4. 内容层单独动

副标题默认透明，hover 后淡入：

```css
.island__subtitle {
  opacity: 0;
}

.island--hovered .island__subtitle {
  opacity: 1;
}
```

这样展开时信息逐步出现，比所有内容同时变化更自然。

## 为什么更丝滑

浏览器渲染大致分为：

```text
样式计算 → 布局 → 绘制 → 合成
```

`width` / `height` 容易触发布局和重绘。

`transform` / `opacity` 更容易只走合成阶段。

所以同样是“变大”，用 `transform` 通常比改宽高更平滑。

## 剩余风险

当前仍然使用：

```css
backdrop-filter
box-shadow
```

它们在透明窗口里仍可能有一定合成成本。如果后续出现卡顿，优先继续降低 blur 和阴影。

## 验证方式

```powershell
pnpm build
pnpm tauri dev
```

肉眼检查：

- hover 展开是否顺滑
- 快速移入移出是否抖动
- 文本是否闪烁
- 透明窗口是否有明显拖影
