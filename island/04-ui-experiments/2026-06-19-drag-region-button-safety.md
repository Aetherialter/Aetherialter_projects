# Drag Region 与按钮点击保护记录

## 结论

本次收窄了 Tauri 拖拽区域，避免 widget 切换按钮被窗口拖拽行为抢走点击。

旧结构：

```html
<section data-tauri-drag-region>
  ...
  <button>Clock</button>
  <button>Mock</button>
</section>
```

这意味着整个岛都是拖拽区域，按钮也在拖拽热区里。

新结构：

```html
<section>
  <div data-tauri-drag-region>dot</div>
  <div data-tauri-drag-region>content</div>
  <div>
    <button>Clock</button>
    <button>Mock</button>
  </div>
</section>
```

按钮不再属于 drag region。

## 修改范围

```text
D:\Aether_island\index.html
D:\Aether_island\src\main.ts
D:\Aether_island\src\styles.css
```

## 为什么要改

Tauri 的 `data-tauri-drag-region` 会把对应区域用于拖动无边框窗口。

如果按钮在这个区域里，真实运行时可能出现：

- 点击按钮变成拖动窗口。
- click 事件不稳定。
- 用户以为按钮失效。

所以交互控件必须从 drag region 中分离出来。

## 事件保护

按钮新增：

```ts
button.addEventListener("pointerdown", (event) => {
  event.stopPropagation();
});
```

并保留：

```ts
button.addEventListener("click", ...)
```

`pointerdown` 阶段先阻止冒泡，可以降低父级拖拽/hover 状态干扰按钮点击的风险。

## CSS 保护

按钮新增：

```css
position: relative;
z-index: 1;
```

确保按钮在当前胶囊内部交互层级更明确。

## 当前交互分区

```text
dot + content
  -> 可以拖动窗口

Clock / Mock buttons
  -> 可以点击切换 active widget
```

这比整块胶囊都可拖动更适合后续复杂交互。

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

- 拖动文字或状态点区域时，窗口是否可拖动。
- 点击 `Clock` / `Mock` 时，是否不会拖动窗口。
- 点击后 active 样式是否变化。
- active widget 是否切换。

## 下一步建议

下一步可以让 Rust 把 widget 列表发送给前端。

这样前端不再硬编码：

```text
Clock
Mock
```

而是根据 runtime 注册的 widget 动态生成切换器。
