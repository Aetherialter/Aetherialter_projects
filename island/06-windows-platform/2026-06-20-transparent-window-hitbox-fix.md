# Aether_island 透明窗口吞鼠标区域修复

## 问题

视觉上的岛只有一个小胶囊，但 Tauri/Win32 窗口本身仍是矩形。

之前窗口固定为：

```text
480 x 120
```

所以即使透明区域看不见，它仍可能占用鼠标命中区域，造成“岛外很大一部分不可用鼠标”的体验。

## 修复策略

前端根据岛状态动态调整真实窗口尺寸：

```text
collapsed: 232 x 44
hovered:   480 x 120
```

折叠态缩小真实窗口矩形，减少透明区域拦截鼠标。

展开态恢复大窗口，为 GitHub 热力图和 widget 切换留出空间。

## 实现位置

`src/tauri-bridge.ts` 新增壳窗口布局策略：

```ts
const SHELL_WINDOW_SIZE = {
  collapsed: { width: 232, height: 44 },
  hovered: { width: 480, height: 120 },
} as const;
```

状态变化时调用：

```ts
applyShellWindowLayout(state)
```

并重新居中到主屏幕顶部。

## 烟测更新

窗口烟测现在接受两种合法尺寸：

```text
232 x 44
480 x 120
```

因为启动或状态切换时可能捕捉到折叠态或展开态。

## 验证结果

已通过：

```powershell
pnpm check
pnpm smoke:window
```

结果：

- Rust tests：24 passed。
- Frontend build：passed。
- Python tests：7 passed。
- Window smoke：passed。
