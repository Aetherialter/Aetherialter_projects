# 前端 IslandSnapshot 契约记录

## 结论

本次为 Aether_island 前端增加了最小状态快照接口：

```ts
type IslandSnapshot = {
  title: string;
  subtitle: string;
  tone: IslandTone;
};
```

它让当前 UI 不再完全依赖硬编码文本，为后续 Rust Runtime / WidgetSnapshot 接入预留了稳定入口。

## 修改范围

```text
D:\Aether_island\index.html
D:\Aether_island\src\main.ts
D:\Aether_island\src\styles.css
```

## 新增 DOM 契约

`index.html` 中为可更新节点增加了稳定 ID：

```html
<div id="island-dot" class="island__dot" aria-hidden="true"></div>
<span id="island-title" class="island__title">Aether_island</span>
<span id="island-subtitle" class="island__subtitle">standby</span>
```

这些 ID 是前端渲染层的临时契约。后续如果接入 Rust event 或 WidgetSnapshot，可以直接更新这些节点。

## 新增 TypeScript 类型

```ts
type IslandState = "collapsed" | "hovered";
type IslandTone = "ready" | "busy" | "warning" | "error";

type IslandSnapshot = {
  title: string;
  subtitle: string;
  tone: IslandTone;
};
```

其中：

- `IslandState` 表示交互状态。
- `IslandTone` 表示语义状态。
- `IslandSnapshot` 表示 UI 可渲染数据。

## 渲染入口

```ts
function renderSnapshot(snapshot: IslandSnapshot): void {
  setText(islandTitle, snapshot.title);
  setText(islandSubtitle, snapshot.subtitle);
  islandDot?.setAttribute("data-tone", snapshot.tone);
}
```

这个函数是后续接入 Runtime 的关键入口。

未来 Rust 可以通过 Tauri event 推送类似结构：

```json
{
  "title": "GitHub",
  "subtitle": "12 contributions",
  "tone": "ready"
}
```

前端只需要调用 `renderSnapshot(snapshot)`。

## CSS 语义状态

新增状态点 tone 样式：

```css
.island__dot[data-tone="busy"]
.island__dot[data-tone="warning"]
.island__dot[data-tone="error"]
```

后续无需改 DOM 结构，只改变 `data-tone` 即可切换状态颜色。

## 工程价值

这一步的价值不是功能复杂，而是边界变清楚：

```text
交互状态：collapsed / hovered
业务状态：ready / busy / warning / error
显示数据：title / subtitle / tone
```

前端开始具备接收外部状态快照的能力。

## 验证结果

以下命令已通过：

```powershell
pnpm build
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
```

## 下一步

后续可以在 Rust 侧定义同构结构：

```rust
pub struct WidgetSnapshot {
    pub title: String,
    pub subtitle: String,
    pub tone: WidgetTone,
}
```

然后通过 Tauri event 推送到前端。
