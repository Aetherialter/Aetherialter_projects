# Collapsed Clock and Switcher Layout Fix

日期：2026-06-19

## 问题

用户指出两个 UI 问题：

1. 常态下岛只显示 `Clock / Mock / Sidecar` 这类组件名称，选中 Clock 后右侧应该直接显示时间。
2. 展开后 `Clock / Mock / Sidecar` 下方或边界处还露出一个 `Clock`，视觉上像文字跑到了岛外。

## 根因

### 1. collapsed 下切换器还占布局空间

`widget-switcher` 之前只是透明：

```css
opacity: 0;
pointer-events: none;
```

但它仍然在 flex 布局里占宽度。这会挤压 `island__content`，导致常态信息展示不自然。

### 2. 内容区和切换器都显示同一类文字

当前数据结构里：

- `island-title` 显示当前 widget 标题，例如 `Clock`。
- `widget-switcher` 里也有 `Clock` 按钮。

展开时两者同时出现，如果布局空间不足，就会像边界处多冒出一个 `Clock`。

## 修改

文件：

```text
D:\Aether_island\src\styles.css
D:\Aether_island\src\main.ts
```

### CSS

collapsed 状态下：

- `widget-switcher` 的 `max-width` 设为 `0`。
- `visibility` 设为 `hidden`。
- `overflow` 设为 `hidden`。
- 内容区改成一行展示标题和摘要。
- 标题与摘要启用 `text-overflow: ellipsis`，避免文字冲出岛边界。

展开状态下：

- `widget-switcher` 恢复最大宽度。
- 按钮只在 hover 展开后参与布局和点击。
- 岛本体 `overflow: hidden`，避免子元素溢出边界。

### TypeScript

浏览器预览模式下，Clock 的 subtitle 从固定 `browser preview` 改为真实本地时间：

```ts
new Intl.DateTimeFormat(undefined, {
  hour: "2-digit",
  minute: "2-digit",
  second: "2-digit",
  hour12: false,
}).format(now)
```

Tauri 真实运行时仍然由 Rust `SystemClockWidget` 提供时间。

## 小白解释

之前的问题像是：虽然按钮被“隐身”了，但它还坐在座位上，占着空间。

这次修改后，常态下按钮不仅隐身，还把座位收起来；展开时再把座位展开。所以常态能更自然地显示：

```text
Clock 19:xx:xx
```

展开时再显示：

```text
Clock 19:xx:xx    Clock Mock Sidecar
```

并且文字不会跑出岛的边界。

## 验证

在 `D:\Aether_island` 运行：

```powershell
pnpm build
pnpm check
pnpm smoke:window
```

结果：

- 前端 TypeScript + Vite build 通过。
- `pnpm check` 通过：Rust 18 测试、clippy、前端 build、Python 7 测试均通过。
- 第一次 `pnpm smoke:window` 被已有 `aether-island` debug 实例阻止。
- 确认残留实例路径为 `D:\Aether_island\src-tauri\target\debug\aether-island.exe` 后关闭。
- 第二次 `pnpm smoke:window` 通过，窗口契约断言通过。

## 剩余风险

Playwright 包存在，但浏览器二进制未安装，因此本轮没有自动截图验收。视觉效果仍建议用户在真实 Tauri 窗口中人工看一下 collapsed/hovered 两种状态。
