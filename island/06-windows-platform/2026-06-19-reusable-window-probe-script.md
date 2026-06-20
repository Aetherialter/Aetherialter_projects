# Reusable Window Probe Script

日期：2026-06-19

## 结论

本轮把临时 Win32 窗口探针沉淀成项目脚本：

```powershell
pnpm probe:window
```

它会查找正在运行的 `aether-island` 进程，枚举该进程的 Win32 窗口，并输出窗口扩展样式 JSON。

这让 v0.4 的真实窗口属性可以重复验证。

## 新增文件

```text
D:\Aether_island\scripts\probe-window.ps1
```

`package.json` 新增：

```json
{
  "scripts": {
    "probe:window": "powershell -NoProfile -ExecutionPolicy Bypass -File ./scripts/probe-window.ps1"
  }
}
```

## 使用方式

先启动真实 Tauri 桌面窗口：

```powershell
pnpm tauri dev
```

另开一个 PowerShell，执行：

```powershell
pnpm probe:window
```

## 本轮验证输出

探针成功输出主窗口：

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

## 探针检查了什么

脚本通过 Win32 API 读取：

- `EnumWindows`
- `GetWindowThreadProcessId`
- `GetWindowTextW`
- `GetWindowLongPtrW(GWL_EXSTYLE)`
- `GetWindowRect`

并解析这些 extended style：

- `WS_EX_TOOLWINDOW`
- `WS_EX_TOPMOST`
- `WS_EX_LAYERED`
- `WS_EX_TRANSPARENT`
- `WS_EX_APPWINDOW`
- `WS_EX_NOACTIVATE`

## 本轮踩坑

PowerShell 脚本的 `param(...)` 必须放在脚本最前面。

第一次写成：

```powershell
$ErrorActionPreference = "Stop"

param(...)
```

会导致：

```text
param : The term 'param' is not recognized
```

修正后：

```powershell
param(...)

$ErrorActionPreference = "Stop"
```

## 验证结果

运行：

```powershell
pnpm probe:window
```

结果：成功输出窗口 JSON。

随后运行：

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

v0.4 的窗口验收现在有固定命令：

```powershell
pnpm probe:window
```

这比临时复制一大段 PowerShell 更可靠，也更适合后续写入 README、发布检查清单或 CI 辅助脚本。
