# Window Probe Assertions

日期：2026-06-19

## 结论

本轮给 `pnpm probe:window` 增加了断言模式：

```powershell
pnpm probe:window:assert
```

普通 `probe:window` 负责输出窗口 JSON；`probe:window:assert` 会在窗口属性不符合 v0.4 当前契约时直接失败。

## 新增命令

`package.json` 新增：

```json
{
  "scripts": {
    "probe:window:assert": "powershell -NoProfile -ExecutionPolicy Bypass -File ./scripts/probe-window.ps1 -AssertAetherIsland"
  }
}
```

## 断言项

当前断言主窗口：

- 存在可见标题为 `Aether_island` 的窗口。
- `ToolWindow = true`
- `TopMost = true`
- `Layered = true`
- `Transparent = false`
- `AppWindow = false`
- `NoActivate = true`
- `Width = 480`
- `Height = 120`

其中 `Transparent = false` 是因为当前默认策略是 `InteractiveOnly`，不启用整窗鼠标穿透。

## 成功输出

本轮执行：

```powershell
pnpm probe:window:assert
```

主窗口输出：

```json
{
  "Title": "Aether_island",
  "ExStyleHex": "0x8080198",
  "ToolWindow": true,
  "TopMost": true,
  "Layered": true,
  "Transparent": false,
  "AppWindow": false,
  "NoActivate": true,
  "Width": 480,
  "Height": 120
}
```

最终输出：

```text
Aether_island window assertions passed.
```

## 失败行为

如果断言失败，脚本会列出具体失败项并以非零退出码结束。

例如：

```text
Aether_island window assertions failed: expected TopMost=true; expected NoActivate=true
```

这让窗口验收从“人工看 JSON”升级成“命令可失败”。

## 验证结果

执行顺序：

```powershell
pnpm tauri dev
pnpm probe:window:assert
```

断言通过。

关闭 Tauri 后执行：

```powershell
pnpm check
```

结果：

- Rust tests：16 个全部通过。
- Frontend build：通过。
- Python tests：6 个全部通过。
- 最终输出 `All checks passed.`

进程检查未发现项目 `aether-island` 或 Vite 残留。

## 对 v0.4 的意义

现在 v0.4 的窗口契约有两类命令：

```powershell
pnpm probe:window
```

用于查看窗口状态。

```powershell
pnpm probe:window:assert
```

用于验收窗口状态。

这比单纯保存文档截图更强，因为它能在未来改动破坏窗口契约时直接失败。
