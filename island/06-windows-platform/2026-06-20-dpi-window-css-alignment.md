# DPI 下窗口尺寸与 CSS 尺寸对齐

## 结论

本次修复把前端 `.island` 的尺寸从固定 `240px x 37px` 改为 `width: 100%; height: 100%;`，让岛的视觉区域始终填满 Tauri 创建的真实窗口区域。

这一步的目标不是改动画，而是先解决更底层的问题：在 Windows DPI 缩放环境下，Rust/Tauri 设置的物理窗口尺寸和 WebView 里的 CSS 像素可能不是同一套坐标单位。如果两边都硬编码尺寸，视觉区域和鼠标交互区域就可能出现错位。

## 问题表现

当前设计要求是：

- 窗口固定在主屏幕正上方。
- 窗口范围就是交互范围。
- 灵动岛视觉范围必须和窗口范围一致。
- 常态只保留 Clock。

但在高 DPI 缩放下，Win32 探针看到的窗口尺寸可能是 `160x25`，而 Rust 端设置的物理尺寸是 `240x37`。这不是窗口真的被随意改小了，而是 Windows 对不同 DPI 感知上下文做了坐标虚拟化。

## 根因

这里有两套“尺子”：

1. Rust/Tauri 窗口层使用物理尺寸：`PhysicalSize { width: 240, height: 37 }`。
2. WebView/CSS 层使用 CSS 像素：以前 `.island` 也写死为 `240px x 37px`。

在 150% DPI 缩放下，`240 / 1.5 = 160`，`37 / 1.5` 约等于 `25`。所以 Win32 探针看到 `160x25` 是符合 DPI 虚拟化现象的。

如果 CSS 继续硬编码 `240px x 37px`，WebView 内部可能尝试绘制一个比可见窗口更大的岛。结果就是用户感受到：

- 鼠标在岛外某些区域也触发交互。
- 视觉边界和实际窗口边界不一致。
- hover 动画看起来像抽搐或边缘误触。

## 设计决策

窗口尺寸由 Rust/Tauri 统一控制，前端 CSS 不再重复声明窗口级尺寸。

当前职责划分：

- Rust/Tauri：决定窗口真实大小、位置、置顶、透明、任务栏隐藏、Win32 扩展样式。
- CSS：只负责在给定窗口内部绘制视觉形态。
- HoverController：只负责 `pointerenter` / `pointerleave` 状态切换。

这符合单一职责原则：窗口边界不要由 Rust 和 CSS 两边同时抢控制权。

## 本次代码变化

文件：

```text
D:\Aether_island\src\styles.css
```

关键变化：

```css
.island {
  width: 100%;
  height: 100%;
}
```

含义：

- 岛的外层容器永远等于 WebView 可用区域。
- WebView 可用区域永远跟随 Tauri 窗口。
- 鼠标交互范围由窗口边界自然约束。

## 验证结果

已执行：

```powershell
pnpm check
pnpm smoke:window
```

结果：

- Rust 测试：15 passed。
- Rust clippy：通过。
- Frontend build：通过。
- Python lint/tests：通过。
- Window smoke：通过。

Win32 探针显示主窗口：

```text
Title       = Aether_island
Top         = 0
ToolWindow  = true
TopMost     = true
Layered     = true
Transparent = false
AppWindow   = false
NoActivate  = true
Size        = 160x25
```

其中 `160x25` 是当前 DPI 下的虚拟化尺寸，脚本已允许 `240x37` 或 `160x25` 两种观测结果。

## 后续风险

当前 smoke 能证明 Win32 层窗口样式和位置正确，但不能完全替代肉眼视觉验证。下一步建议用真实运行窗口检查：

- 岛是否正好贴在屏幕顶部中央。
- 鼠标在岛外是否完全不触发 hover。
- hover 动画是否还能被边缘快速移动触发抖动。
- 视觉阴影是否只存在胶囊内部，没有窗口外扩散。

如果仍有边缘抖动，下一轮应重点检查 `transform: scale(...)` 是否让视觉形态缩小后暴露出窗口内透明边缘，从而造成“看起来不在岛上，但仍然属于窗口”的交互错觉。
