# Tauri 里选择 TypeScript / JavaScript 是否轻量

## 结论

可以。对于 Tauri 来说，选择 `TypeScript / JavaScript` 这一类前端方案仍然可以保持很轻量，尤其是配合 `Vanilla` 模板时。

真正决定运行时重量的，不是“TypeScript 还是 JavaScript”这个选项本身，而是：

- 是否引入大型前端框架。
- 是否引入很重的状态管理和 UI 运行时。
- 是否把大量逻辑堆在前端。

如果目标是 IslandGate 这种小而轻的桌面灵动岛，`Vanilla + JavaScript` 最轻；`Vanilla + TypeScript` 也依然很轻，只是开发期多一层类型检查。

## 结论先讲清楚

### 1. 运行时几乎不因为 TypeScript 变重

TypeScript 在本质上会被编译成 JavaScript。也就是说：

```text
TypeScript = 开发时有类型
运行时还是 JavaScript
```

所以它增加的主要是开发期的检查成本，不是运行时成本。

### 2. 真正影响轻量化的是前端框架

以下做法通常更轻：

- 原生 HTML / CSS / JS
- 少量事件绑定
- 少量状态更新

以下做法通常更重：

- React / Vue / Svelte 之类框架
- 很多抽象组件层
- 大量第三方包
- 复杂全局状态管理

## 对 IslandGate 的建议

### 如果你最在乎极简

选：

```text
TypeScript / JavaScript
UI template: Vanilla
UI flavor: JavaScript
```

这样前端运行时最朴素，适合灵动岛这种小 UI。

### 如果你更在乎后续维护

选：

```text
TypeScript / JavaScript
UI template: Vanilla
UI flavor: TypeScript
```

这样依然轻量，但你写 Rust 后端命令、Tauri 事件、前端状态对象时，类型提示会更稳，后面不容易把事件字段写错。

## Rust 视角下怎么理解

从 Rust 的思路看，前端语言选型要区分两件事：

- **开发体验**：TypeScript 更容易发现字段不一致、事件结构不一致。
- **运行性能**：差别很小，因为最后都要落到 WebView 执行的 JavaScript。

所以如果你担心“TypeScript 会拖慢应用”，通常不用太担心。

更该担心的是：

- 前端包是否过大。
- 是否引入不必要的框架。
- 是否在前端做了过多数据处理。

## 工程取舍

### JavaScript 的优点

- 最直接。
- 少一层类型工具链。
- 学习和初始化更轻。

### TypeScript 的优点

- 事件和数据结构更稳。
- 更适合 Rust 后端配合。
- 后期维护成本通常更低。

### 对你的项目的平衡建议

如果你现在更想先跑起来：

```text
JavaScript
```

如果你现在已经确定后面会持续加组件、加事件、加状态：

```text
TypeScript
```

两者都可以很轻量，区别主要在开发体验，不在运行时重量。

## 验证方式

你后面可以用这些方式保持轻量：

- 只用 `Vanilla` 模板。
- 不引入大框架。
- 不引入多余前端依赖。
- 只保留必要的状态和事件。
- Rust 负责数据和系统逻辑，前端只负责展示。

## 简历与面试表达

可以这样说：

```text
项目前端采用 Tauri 的 Vanilla 模板，以原生 HTML/CSS/JavaScript 或 TypeScript 构建轻量 UI。TypeScript 仅用于开发期类型约束，不增加显著运行时负担，运行时仍由系统 WebView 承载的 JavaScript 执行，从而兼顾轻量化和工程可维护性。
```

## 参考来源

- Tauri Create Project：https://v2.tauri.app/start/create-project/
- Tauri Frontend Configuration：https://v2.tauri.app/start/frontend/
- Tauri Architecture：https://v2.tauri.app/concept/architecture/
