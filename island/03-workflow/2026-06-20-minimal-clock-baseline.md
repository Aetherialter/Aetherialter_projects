# Aether_island 最小 Clock 基线收束

## 结论

本次按硬性要求将项目收束为最小可控版本：

- 只保留默认 Clock。
- 删除 GitHub Heatmap。
- 删除 GitHub 相关 Rust/TS 代码。
- 删除 mock widget。
- 删除 sidecar widget 与 sidecar Rust 协议模块。
- 删除前端 widget 切换按钮。
- 删除前端详情区域。
- 真实窗口尺寸固定等于岛本体范围。

## 当前交互范围

窗口真实尺寸：

```text
292 x 44
```

岛视觉尺寸：

```text
292 x 44
```

这意味着：

```text
岛的范围 = Win32/Tauri 窗口范围 = 鼠标交互范围
```

透明窗口外不再有大面积鼠标拦截区。

## 当前 UI

只显示：

```text
Clock    HH:MM:SS
```

没有：

- system clock 按钮。
- github 按钮。
- mock 按钮。
- sidecar 按钮。
- GitHub 热力图。
- widget 详情区。

## 当前 Rust Runtime

`NativeWidgetProvider` 已收束为单 widget provider：

```rust
pub struct NativeWidgetProvider {
    widget: Box<dyn IslandWidget>,
}
```

当前只注册：

```rust
create_system_clock_widget()
```

## 验证结果

已通过：

```powershell
pnpm check
pnpm smoke:window
```

结果：

- Rust tests：15 passed。
- Rust clippy：passed。
- Frontend build：passed。
- Python lint/tests：passed。
- Window smoke：passed。

烟测确认主窗口：

```text
Width=292
Height=44
ToolWindow=true
TopMost=true
Layered=true
Transparent=false
AppWindow=false
NoActivate=true
```

## 后续原则

后续再加任何 widget，都必须先解决“交互范围不能超过岛范围”的约束。

也就是说，不允许再通过扩大透明窗口来放内容，除非同时实现明确的窗口尺寸切换、鼠标穿透或区域命中控制策略。
