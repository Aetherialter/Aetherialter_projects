# Tauri 窗口顶部居中定位：API 版本校验记录

## 结论

“API 是否符合当前版本”不能靠猜，要以本地安装的类型定义和构建结果为准。

本次 `@tauri-apps/api` 的实际类型要求是：

```ts
import {
  getCurrentWindow,
  PhysicalPosition,
  primaryMonitor,
} from "@tauri-apps/api/window";
```

其中：

- `primaryMonitor` 是模块函数，不是 `Window` 实例方法。
- `setPosition` 不接受普通 `{ x, y }` 对象。
- `setPosition` 需要 `new PhysicalPosition(x, y)`。

## 原错误

错误 1：

```text
Property 'primaryMonitor' does not exist on type 'Window'.
```

说明当前版本里不能写：

```ts
const monitor = await appWindow.primaryMonitor();
```

应该写：

```ts
const monitor = await primaryMonitor();
```

错误 2：

```text
Argument of type '{ x: number; y: number; }' is not assignable to parameter of type 'LogicalPosition | PhysicalPosition | Position'.
```

说明不能写：

```ts
await appWindow.setPosition({ x, y });
```

应该写：

```ts
await appWindow.setPosition(new PhysicalPosition(x, y));
```

## 当前实现

```ts
async function positionWindow(): Promise<void> {
  const appWindow = getCurrentWindow();
  const monitor = await primaryMonitor();

  if (!monitor) {
    return;
  }

  const windowSize = await appWindow.outerSize();
  const x = Math.round(
    monitor.position.x + (monitor.size.width - windowSize.width) / 2,
  );
  const y = monitor.position.y + 12;

  await appWindow.setPosition(new PhysicalPosition(x, y));
}
```

## 为什么要加 monitor.position

多显示器环境下，主显示器左上角不一定是 `(0, 0)`。

因此横向居中不能只算：

```ts
(monitor.size.width - windowSize.width) / 2
```

还要加上：

```ts
monitor.position.x
```

纵向顶部也类似，使用：

```ts
monitor.position.y + 12
```

## 权限配置

`src-tauri/capabilities/default.json` 中需要允许：

```json
"core:window:allow-outer-size",
"core:window:allow-primary-monitor",
"core:window:allow-set-position"
```

## 验证结果

以下命令已通过：

```powershell
pnpm build
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
```

## 人工验收项

仍需运行：

```powershell
pnpm tauri dev
```

人工检查：

- 窗口是否在主屏幕顶部水平居中。
- 顶部偏移是否合适。
- hover 动画是否仍顺滑。
- 透明无边框配置是否仍生效。
