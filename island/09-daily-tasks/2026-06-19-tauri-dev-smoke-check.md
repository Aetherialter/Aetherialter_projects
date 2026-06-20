# Tauri Dev Smoke Check

日期：2026-06-19

## 结论

本轮执行了 `pnpm tauri dev` 运行时烟测。结果表明：

- Vite dev server 能启动。
- Rust/Tauri dev profile 能编译完成。
- `aether-island.exe` 能被 Tauri CLI 拉起。
- 观察约 10 秒没有出现立即崩溃输出。

这证明当前代码不只是能 `pnpm build`，也能进入 Tauri 桌面运行路径。

## 执行命令

在 `D:\Aether_island` 运行：

```powershell
pnpm tauri dev
```

关键输出：

```text
VITE v6.4.3 ready in 184 ms
Local: http://localhost:1420/
Compiling aether-island v0.1.0
Finished `dev` profile [unoptimized + debuginfo] target(s) in 2.83s
Running `target\debug\aether-island.exe`
```

随后观察约 10 秒，终端没有出现新的崩溃日志。

## 退出说明

最后通过 Ctrl+C 主动停止 dev 会话，终端显示：

```text
STATUS_CONTROL_C_EXIT
```

这是手动中断导致的退出状态，不代表应用自行崩溃。

## 已验证

1. Tauri CLI 能执行 `beforeDevCommand`。
2. Vite dev server 能提供前端页面。
3. Rust 后端能在 dev profile 下编译通过。
4. Tauri 桌面进程能启动并保持运行一小段时间。

## 未完全验证

这次烟测没有完成肉眼交互验收，仍需人工确认：

1. 胶囊是否真正固定在屏幕正上方。
2. hover 展开/折叠动画是否符合预期。
3. Clock / Mock / Sidecar 切换按钮是否都能点击。
4. 点击 widget 后内容是否立即切换。
5. 无边框、透明背景、无任务栏入口是否在当前 Windows 环境下符合预期。
6. 鼠标穿透策略是否符合当前设计。

## 对 v0.4 的意义

v0.4 需要的不只是静态构建成功，还要证明 Tauri 运行路径能走通。本轮烟测把“能启动”这个风险先压下去。

下一步应该进入人工 UI 验收或自动化窗口截图验收。如果发现视觉或交互问题，再针对 UI 层修正。
