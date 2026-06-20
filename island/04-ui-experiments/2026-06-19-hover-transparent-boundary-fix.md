# Hover Transparent Boundary Fix

日期：2026-06-19

## 用户定位

用户通过手动测试发现：

> 在设定的边界，好像在岛外面，来回移动鼠标会导致小部分抽搐。

这个定位是准确的。

## 根因

上一轮为了稳定动画，把结构拆成：

```text
.island          固定外层命中盒
.island__surface 视觉胶囊，负责 scale 动画
```

这解决了“视觉胶囊缩放导致 hover hitbox 跟着动”的问题。

但它也引入了新的边界问题：

- `.island` 外层命中盒是完整 `292x44`。
- collapsed 可见胶囊是 `.island__surface` 缩放后的视觉区域，大约只有外层的 73%。
- 所以有一圈透明区域“看起来在岛外”，但实际上仍属于 `.island`。
- 鼠标在这圈透明边界来回移动时，会触发 hover 逻辑，造成小范围抽搐。

## 修改

文件：

```text
D:\Aether_island\src\main.ts
D:\Aether_island\src-tauri\src\lib.rs
```

### 前端 hover 判定

不再用外层 `.island` 的 `mouseenter` 直接展开。

改为：

1. 监听 `pointermove`。
2. 记录鼠标坐标。
3. collapsed 状态下，只有鼠标进入当前可见 `.island__surface` 附近才展开。
4. pointerleave 时，用外层 `.island` 加安全余量判断是否真的需要收起。

核心思想：

```text
进入：按可见胶囊判断
离开：按安全区域判断
```

这避免透明边界触发展开，同时又避免刚离开一点就立刻收起。

### Rust 启动初始化

由于前端不再启动时主动发送 collapsed 状态，Rust setup 阶段主动应用一次 collapsed click-through 样式：

```text
ShellState::Collapsed + ClickThroughPolicy::InteractiveOnly
```

这样 `WS_EX_LAYERED` 不依赖前端 hover/状态同步，启动窗口样式更稳定。

## 小白解释

之前像是岛外面有一圈透明按钮。你看不到它，但鼠标碰到它，程序会以为你碰到了岛。

现在改成：

- 鼠标进入时，看它有没有碰到“看得见的胶囊”。
- 鼠标离开时，允许有一点安全缓冲，避免太敏感。

这样你在透明外圈来回蹭，就不应该再触发展开/收起。

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
- 窗口探针确认主窗口：
  - `ToolWindow=true`
  - `TopMost=true`
  - `Layered=true`
  - `Transparent=false`
  - `AppWindow=false`
  - `NoActivate=true`

## 剩余风险

这轮修复针对用户明确定位的透明边界问题。如果仍能复现，下一步应把 `collapsedHitPadding` 继续缩小，或者把 `.island` 外层尺寸改成更贴近 collapsed 可见胶囊，再通过一个单独的 invisible safe-zone 控制离开 hysteresis。
