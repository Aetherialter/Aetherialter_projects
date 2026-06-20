# Hover 展开态 Widget 切换入口记录

## 结论

本次在前端 hover 展开态新增一个极简 widget 切换入口：

```text
Clock | Mock
```

点击后调用 Rust command：

```ts
invoke("set_active_widget", { widgetId })
```

这样可以从 UI 层真实验证 `active_widget_id` 选择策略。

## 修改范围

```text
D:\Aether_island\index.html
D:\Aether_island\src\main.ts
D:\Aether_island\src\styles.css
```

## HTML 结构

新增：

```html
<div class="island__switcher">
  <button data-widget-id="system-clock">Clock</button>
  <button data-widget-id="mock">Mock</button>
</div>
```

它位于胶囊内部，靠右显示。

## TypeScript 行为

新增状态：

```ts
let activeWidgetId = "system-clock";
```

新增函数：

```ts
setActiveWidget(widgetId)
```

它会：

1. 更新本地 active 按钮样式。
2. 调用 Rust command。
3. 如果调用失败，只输出 warning，不阻断 UI。

## CSS 行为

切换入口默认不可见：

```text
opacity: 0
pointer-events: none
```

hover 展开后可见：

```text
opacity: 1
pointer-events: auto
```

这保证 collapsed 状态仍然保持极简。

## 为什么这是开发态入口

当前只有两个 widget：

```text
Clock
Mock
```

这个切换器主要用于验证 runtime 能力，而不是最终产品 UI。

后续正式交互可以改成：

- 图标按钮。
- 托盘菜单。
- 快捷键。
- 展开态卡片。
- 用户配置文件。

## 当前风险

Tauri command 参数从前端传入：

```ts
{ widgetId }
```

Rust command 参数是：

```rust
widget_id: Option<String>
```

Tauri 通常会做 camelCase 到 snake_case 的命令参数映射，但真实运行仍建议通过：

```powershell
pnpm tauri dev
```

手动点击验证。

如果运行时报参数缺失，可以改成显式对象结构或调整 command 参数名。

## 验证结果

以下命令已通过：

```powershell
pnpm build
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
```

## 人工验收

运行：

```powershell
pnpm tauri dev
```

检查：

- hover 后是否显示 `Clock | Mock`。
- 点击 `Mock` 后是否切到 `refresh #n`。
- 点击 `Clock` 后是否回到 `local HH:MM:SS`。
- 胶囊文本和按钮是否不重叠。

## 下一步建议

下一步可以让 Rust 主动把 widget 列表发给前端。

这样前端不再硬编码：

```text
Clock
Mock
```

而是从 runtime 动态渲染 widget 切换器。
