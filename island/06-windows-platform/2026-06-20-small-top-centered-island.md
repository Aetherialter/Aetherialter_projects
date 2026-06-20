# Aether_island 小尺寸顶部居中定位

## 结论

本次将岛缩小，并把窗口定位逻辑下沉到 Rust 初始化阶段。

当前设计尺寸：

```text
240 x 37
```

当前位置：

```text
主屏幕顶部居中
Top = 0
```

## 为什么定位放到 Rust

之前前端通过 Tauri JS API 设置窗口位置和尺寸，但在 Windows DPI 缩放环境下，JS 侧尺寸会出现逻辑像素和物理像素换算差异。

现已删除前端窗口布局兜底，改为 Rust 在窗口初始化阶段统一处理：

```rust
window.set_size(...)
window.set_position(...)
```

这样窗口创建后就位于顶部居中，不依赖 WebView 前端脚本加载时机。

## DPI 烟测说明

在当前 Windows 缩放环境下，Win32 探针可能读到：

```text
160 x 25
```

这对应 150% DPI 缩放下的虚拟化读数。

因此烟测接受两种尺寸：

```text
240 x 37   物理尺寸
160 x 25   DPI 虚拟化尺寸
```

同时强制断言：

```text
Top <= 8
```

## 验证结果

已通过：

```powershell
pnpm check
pnpm smoke:window
```

烟测确认：

```text
Top=0
ToolWindow=true
TopMost=true
Layered=true
AppWindow=false
NoActivate=true
```
