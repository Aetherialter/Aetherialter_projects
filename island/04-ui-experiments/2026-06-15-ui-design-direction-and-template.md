# Aether_island UI 设计方向与模板代码

## 结论

当前最适合 Aether_island 的设计方向是：

```text
透明 HUD + 低侵入胶囊 + hover 展开
```

它比普通卡片更轻，比完整面板更不打扰。你可以把它理解成：

- 平时只露出很小的一块状态条
- 鼠标移上去时再展开更多信息
- 颜色尽量低饱和，不抢当前软件的视觉注意力

这条路线适合桌面常驻组件，因为它先解决“别碍事”，再解决“好看”。

## 设计原则

### 1. 先轻，再美

第一版不要追求复杂内容。先让窗口像一个安静的桌面状态浮层，再慢慢加信息量。

### 2. 使用低对比配色

推荐默认配色：

- 背景：深色半透明
- 文字：偏灰白
- 状态点：低饱和绿或蓝
- 强调色：只在 hover 或告警时出现

### 3. 保持边界清楚

组件要有自己的形状感，但不要像一块很重的卡片压在桌面上。

推荐用：

- 圆角胶囊
- 很轻的阴影
- 很淡的边线，或者干脆不显示边线
- 透明窗口背景

### 4. 动画只做“质感”，不要做“炫技”

你这个项目的动画任务不是吸睛，而是让展开和收起更顺。

推荐动画曲线：

```css
transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
```

## 可学习模板

下面是一套最小模板。你可以直接把它当作学习底稿，然后替换颜色、尺寸和文案。

### `index.html`

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Aether_island</title>
    <link rel="stylesheet" href="/src/styles.css" />
  </head>
  <body>
    <main id="app" class="app-shell">
      <section
        id="island"
        class="island island--collapsed"
        data-tauri-drag-region
        aria-label="Aether island status"
      >
        <div class="island__dot" aria-hidden="true"></div>
        <div class="island__content">
          <span class="island__title">Aether_island</span>
          <span class="island__subtitle">standby</span>
        </div>
      </section>
    </main>

    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

### `src/main.ts`

```ts
const island = document.querySelector<HTMLElement>("#island");

type IslandState = "collapsed" | "hovered";

function setIslandState(state: IslandState): void {
  if (!island) {
    return;
  }

  island.classList.toggle("island--collapsed", state === "collapsed");
  island.classList.toggle("island--hovered", state === "hovered");
}

window.addEventListener("DOMContentLoaded", () => {
  setIslandState("collapsed");

  island?.addEventListener("mouseenter", () => {
    setIslandState("hovered");
  });

  island?.addEventListener("mouseleave", () => {
    setIslandState("collapsed");
  });
});
```

### `src/styles.css`

```css
:root {
  font-family:
    Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  color: rgba(255, 255, 255, 0.92);
  background: transparent;
  font-synthesis: none;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

* {
  box-sizing: border-box;
}

html,
body,
#app {
  width: 100%;
  height: 100%;
  margin: 0;
  background: transparent;
}

body {
  overflow: hidden;
}

.app-shell {
  width: 100%;
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: flex-start;
  padding-top: 18px;
}

.island {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  width: 160px;
  height: 32px;
  padding: 0 12px;
  border-radius: 999px;
  background: rgba(18, 19, 24, 0.86);
  box-shadow:
    0 12px 30px rgba(0, 0, 0, 0.24),
    inset 0 1px 0 rgba(255, 255, 255, 0.08);
  backdrop-filter: blur(18px);
  user-select: none;
  transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}

.island--hovered {
  width: 220px;
  height: 44px;
}

.island__dot {
  width: 7px;
  height: 7px;
  border-radius: 999px;
  background: #72f0b8;
  box-shadow: 0 0 14px rgba(114, 240, 184, 0.72);
}

.island__content {
  display: flex;
  flex-direction: column;
  min-width: 0;
  line-height: 1;
}

.island__title {
  font-size: 12px;
  font-weight: 600;
}

.island__subtitle {
  margin-top: 2px;
  font-size: 10px;
  color: rgba(255, 255, 255, 0.54);
}
```

## 你怎么改成自己的风格

你可以先只改这几项：

- `width / height`：改形状大小
- `border-radius`：改更圆还是更硬朗
- `background`：改更透明还是更厚
- `box-shadow`：改更轻或更立体
- `dot` 的颜色：改成你喜欢的状态色
- `subtitle`：改成 GitHub、Pomodoro、System 等状态文本

## 推荐练习顺序

1. 先把模板原样跑起来。
2. 只改颜色，不改结构。
3. 再改尺寸，不改动画。
4. 最后改内容文案和状态布局。

这样你会清楚知道每个改动会影响什么。

## 下一步建议

如果你愿意，我下一步可以直接给你一版：

```text
三个风格方案
1. 极简状态条
2. 透明胶囊
3. 低调 HUD 面板
```

你可以从里边挑一个作为 Aether_island 的视觉起点。
