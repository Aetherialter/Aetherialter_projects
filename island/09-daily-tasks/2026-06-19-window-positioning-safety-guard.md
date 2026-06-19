# 2026-06-19 窗口定位安全兜底记录

## 结论

本次为 Aether_island 顶部居中定位逻辑增加了错误兜底。

窗口定位失败时，应用不会因为未处理的 Promise rejection 影响 UI 初始化，而是通过 `console.warn` 输出诊断信息。

## 修改范围

```text
D:\Aether_island\src\main.ts
```

## 修改内容

新增安全包装函数：

```ts
function positionWindowSafely(): void {
  void positionWindow().catch((error: unknown) => {
    console.warn("Failed to position Aether_island window.", error);
  });
}
```

启动时改为调用：

```ts
positionWindowSafely();
```

## 为什么需要

窗口定位依赖 Tauri window API 和 capability 权限：

```text
primaryMonitor()
outerSize()
setPosition()
```

如果权限缺失、显示器信息获取失败，或运行环境暂时不支持这些调用，直接 `void positionWindow()` 可能产生未处理的异步错误。

用 `positionWindowSafely()` 后，定位失败不会阻断胶囊 UI 初始化。

## 验证结果

以下命令已通过：

```powershell
pnpm build
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
```

## 未验证项

仍需人工运行：

```powershell
pnpm tauri dev
```

人工检查：

- 窗口是否定位到主屏幕顶部居中。
- 透明无边框是否仍然生效。
- hover 动画是否仍然流畅。
- DevTools console 是否没有定位错误。

## 下一步

如果顶部居中和透明浮层效果稳定，下一步可以进入 Win32 鼠标穿透设计。
