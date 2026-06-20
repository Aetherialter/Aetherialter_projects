# Tauri WebView Interaction Screenshot

日期：2026-06-19

## 结论

本轮对真实 Tauri WebView 做了系统截图级交互验收。

已验证：

- `pnpm tauri dev` 能启动真实桌面窗口。
- `pnpm probe:window` 能获取窗口坐标与 Win32 样式。
- 系统截图能捕获 Aether_island 浮窗。
- 鼠标移动到岛区域后，真实 Tauri WebView 中能看到 hover 展开状态。
- 点击 widget 区域后，真实 Tauri WebView 中显示了 `Mock`，证明 widget 切换链路在 Tauri 容器中生效。

## 截图证据

截图保存位置：

```text
D:\Users\Aetherialter\Documents\Aetherialter_projects\island\04-ui-experiments
```

文件：

- `2026-06-19-tauri-initial.png`
- `2026-06-19-tauri-hover.png`
- `2026-06-19-tauri-click-mock.png`
- `2026-06-19-tauri-wide-hover.png`

其中 `2026-06-19-tauri-wide-hover.png` 显示真实 Tauri 窗口中标题已经切换为 `Mock`。

## 验收过程

启动真实 Tauri：

```powershell
pnpm tauri dev
```

读取窗口探针：

```powershell
pnpm probe:window
```

主窗口返回：

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
  "Left": 613,
  "Top": 8,
  "Width": 480,
  "Height": 120
}
```

随后使用 Windows 截图 API：

- `System.Drawing.Bitmap`
- `Graphics.CopyFromScreen`

裁剪屏幕顶部区域并保存 PNG。

## 本轮发现的边界

透明 Tauri 窗口、DPI 缩放、WebView 内容区域之间存在坐标偏差。

也就是说：

```text
Win32 探针得到的是外层窗口矩形。
系统截图看到的是实际屏幕像素。
WebView 内部胶囊内容并不一定填满整个 480x120 窗口。
```

因此，较小裁剪图可能只截到岛的一部分，容易误判。

本轮扩大截图范围后，能看到真实 Tauri WebView 里的 `Mock` 状态。

## 验证结果

关闭 Tauri dev 后运行：

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

这一步补上了浏览器预览无法证明的部分：真实 Tauri WebView 中 widget 切换确实能显示到窗口里。

当前 UI 证据链：

1. 浏览器预览验证布局和本地 preview widgets。
2. Tauri dev smoke 验证真实应用能启动。
3. Win32 probe 验证窗口样式。
4. 系统截图验证真实 Tauri WebView 中能显示并切到 `Mock`。

## 剩余风险

仍需人工确认：

- 任务栏是否真正不显示。
- Alt-Tab 是否真正不显示。
- hover 展开动画是否符合主观体验。
- 鼠标穿透策略是否符合用户日常使用习惯。
