# Tauri UI flavor 为什么选 TypeScript

## 结论

在 Aether_island 里，UI flavor 建议选：

```text
TypeScript
```

## 原因

### 1. 不增加明显运行时负担

TypeScript 最终会编译成 JavaScript。也就是说：

```text
开发期有类型
运行时还是 JS
```

所以它不会把 Tauri 前端显著变重。

### 2. 适合 Rust 后端配合

这个项目后面会有很多结构化数据：

- Widget snapshot
- Shell event
- Cache record
- Config schema
- IPC message

TypeScript 能在开发时更早发现字段写错、事件结构不一致、状态对象漏字段这些问题。

### 3. 组件一多，类型约束更值钱

现在只是灵动岛壳子，但后面你大概率会不断加组件。

组件越多，事件越多，状态越多，越需要一种方式帮你减少低级错误。TypeScript 正好适合这个场景。

## 为什么不是 JavaScript

JavaScript 当然也能用，而且会更直接。

但如果你后面频繁在 Rust 和前端之间传结构化数据，纯 JS 更容易出现：

- 字段名打错。
- 事件 payload 结构不统一。
- 快照结构更新时漏改前端。

对 Aether_island 这种项目，TS 更稳。

## 轻量性说明

选 TypeScript 不等于选了重前端。

真正决定重量的是：

- 有没有大框架。
- 有没有复杂状态管理。
- 有没有大量第三方包。

只要你继续用 `Vanilla + TypeScript`，它仍然很轻。

## 建议口径

当前选择：

```text
Vanilla + TypeScript
```

这个组合适合：

- 轻量。
- 可维护。
- 事件结构清晰。
- 后面扩展小组件时更稳。
