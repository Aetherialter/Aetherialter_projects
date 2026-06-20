# Aether_island Notes

项目目录：`D:\Aether_island`

笔记目录：`D:\Users\Aetherialter\Documents\Aetherialter_projects\island`

## 目录

- `00-meta`：协作规则
- `01-tauri-basics`：Tauri 框架与入口机制
- `02-setup-decisions`：初始化选择与取舍
- `03-workflow`：Git、分支、审查工作流
- `04-ui-experiments`：UI 实验与前端示例
- `05-troubleshooting`：报错与踩坑排查
- `06-windows-platform`：Win32、窗口、穿透
- `07-architecture-references`：架构横评
- `08-runtime-widgets`：Widget/Runtime 设计
- `09-daily-tasks`：每日任务单

## 当前项目状态

- 分支：`feat/island-shell-foundation`
- 状态：Tauri 透明窗口、胶囊 UI、Rust 到前端 snapshot 事件链、前端 shell state 回传 Rust runtime、Win32 鼠标穿透封装与策略保护、Rust core widget/provider/runtime 生命周期与周期刷新骨架，SystemClockWidget 本地时钟组件，WidgetSnapshot 优先级与 active widget 选择，动态 widget 列表驱动的 hover 切换器，drag region 与按钮点击保护，Sidecar-ready IPC 边界，Python daemon stdio JSON 协议草图与测试，Rust provider/runtime 核心调度单元测试，FrontendSnapshot 完整字段透传，active widget 点击后即时刷新，Tauri dev 运行时烟测通过，浏览器预览模式与 UI 截图验收，Win32 exstyle 探针验证 tool/topmost/layered/noactivate 并移除 app window 样式，窗口样式 bit 规则单元测试，统一 `pnpm check` 质量门禁，可复用 `pnpm probe:window` 原生窗口探针，`pnpm probe:window:assert` 窗口契约断言，真实 Tauri WebView 截图验证 Mock 切换显示
- 初始提交：`chore: initialize tauri project`
- 当前目标：向 v0.4 推进，先完成 Rust-first Widget Runtime，再预留 Python sidecar 接入点
