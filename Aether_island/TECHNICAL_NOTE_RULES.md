# IslandGate 技术讲解沉淀规则

## 1. 规则目标

后续围绕 IslandGate 的技术性问题，默认不仅在对话中解释，还要沉淀为 Markdown 文档，保存到：

```text
D:\Users\Aetherialter\Documents\Aetherialter_projects\island
```

该规则用于把零散问答积累成可复习、可迁移、可用于项目展示和面试表达的技术档案。

## 2. 适用范围

以下内容默认需要生成或更新 Markdown 技术笔记：

- Rust 所有权、生命周期、trait、async、错误处理。
- Tauri、WebView、系统托盘、窗口生命周期。
- Windows Win32 API、窗口样式、鼠标穿透、置顶、Mica/Acrylic。
- 架构边界、模块划分、Provider/Widget/Sidecar 设计。
- 缓存、配置、事件协议、状态快照。
- Python Sidecar、双进程生命周期、IPC、心跳、孤儿进程处理。
- 代码审查中出现的关键技术问题。
- 预发布检查、发布清单、工程质量门禁。

简单状态同步、纯命令回显、非技术闲聊不需要自动生成文档。

## 3. 输出风格

技术笔记参考 `EXPERT_QA_MODE.md` 的要求，默认遵循：

- 先给结论，再展开依据。
- 使用 Markdown 结构化组织。
- 对关键判断说明验证方式或工程依据。
- 对 Rust 问题补充所有权、生命周期、异步、错误处理和 unsafe 安全边界。
- 对 Windows 底层问题补充系统调用、窗口句柄、资源生命周期和平台限制。
- 对工程问题补充可执行 PowerShell 命令、测试方式和风险边界。
- 对可展示内容补充简历/面试表达。

## 4. 文件命名

默认使用稳定、可排序的文件名：

```text
YYYY-MM-DD-topic-slug.md
```

示例：

```text
2026-06-15-rust-first-sidecar-ready-architecture.md
2026-06-15-win32-click-through-window.md
2026-06-15-tauri-project-initialization.md
```

如果同一天同主题需要更新，优先更新已有文件；如果主题明显不同，再创建新文件。

## 5. 推荐文档结构

```markdown
# 标题

## 结论

## 背景

## 核心机制

## Rust 知识点

## Windows / 系统层知识点

## 工程取舍

## 验证方式

## 常见坑

## 简历与面试表达
```

不相关的小节可以省略，但不能用空泛占位内容填充。

## 6. 与代码实现的边界

当前协作方式为：

- 用户负责实现代码。
- Codex 负责审查、预发布、文档和技术解释沉淀。

除非用户明确要求“直接改”“帮我实现”“更新这个文件”，否则 Codex 不主动修改项目源码。

技术笔记、README、ADR、发布清单属于文档工作，可以在用户提出技术解释、审查、预发布或文档整理需求时生成。

## 7. 验收标准

每次生成技术笔记后，需要满足：

- 文档保存到指定 island 目录。
- 文件名清晰可检索。
- 内容与当前项目决策一致。
- 命令示例使用 PowerShell。
- 不包含空泛占位词。
- 不声称尚未实现的功能已经完成。
