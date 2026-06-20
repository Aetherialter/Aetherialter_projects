# Hover Jitter Hitbox Fix

日期：2026-06-19

## 问题

用户反馈：鼠标滑过灵动岛时会抽搐，怀疑是动画太快导致。

## 判断

这类问题不一定是动画速度太快。更常见的根因是 hover 命中区域不稳定：

- collapsed 状态下岛使用 `transform: scaleX(0.73) scaleY(0.73)`。
- hovered 状态下岛变为 `transform: scale(1)`。
- 鼠标进入后元素开始放大或改变布局。
- 元素的视觉边界和鼠标命中区域在动画过程中变化。
- 鼠标可能瞬间被判定为离开，再触发 collapsed。
- collapsed 又缩小，鼠标再次进入，于是来回抽搐。

所以问题表面像“动画快”，本质更像“hover hitbox 跟着动画抖”。

## 修改

文件：

```text
D:\Aether_island\index.html
D:\Aether_island\src\styles.css
D:\Aether_island\src\main.ts
```

### 结构拆分

把原来的单层 `.island` 拆成：

```html
<section id="island" class="island island--collapsed">
  <div class="island__surface">
    ...视觉内容...
  </div>
</section>
```

职责变成：

- `.island`：固定宽高，只负责鼠标命中区域。
- `.island__surface`：负责视觉缩放、背景、阴影、弹性动画。

这样鼠标 hover 绑定在稳定外层，视觉层怎么缩放都不会改变外层命中盒。

### 收起防抖

`mouseleave` 后的 collapsed 延迟从 `80ms` 调整为 `180ms`。

这个不是为了让动画变慢，而是给鼠标在边界附近移动留出缓冲，避免刚离开边界就立刻触发收缩。

## 小白解释

之前像是你站在自动门边上，门一开一关，门框的位置也跟着动，于是系统一会儿觉得你进来了，一会儿觉得你出去了。

现在做法是：

- 门框固定不动。
- 里面的门板做动画。

所以鼠标判断更稳定，视觉动画仍然可以保留弹性。

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
- `pnpm smoke:window` 通过。
- 原生窗口仍保持 `ToolWindow=true`、`TopMost=true`、`Layered=true`、`NoActivate=true`、`AppWindow=false`。

## 剩余风险

自动化烟测能证明窗口启动和样式没有坏，但 hover 手感最终仍需要真实鼠标人工验证。如果仍有轻微抖动，下一步应继续增加 hover hysteresis，例如扩大外层命中盒高度或改用 pointer tracking 判断鼠标是否仍在安全区域内。
