# 为什么 Tauri 初始化不一定优先选 Bun

## 结论

**Bun 可以选，而且 Tauri 官方已经支持。**

但在 Aether_island 这种刚初始化、目标是先把桌面壳跑稳的项目里，我通常不会把 Bun 作为第一推荐，原因不是“不支持”，而是它不是最保守的起步选择。

## 先说官方事实

Tauri 官方的创建项目文档和开发文档都已经列出 `bun` 作为可用命令：

- `bun create tauri-app`
- `bun tauri dev`
- `bun tauri build`

这说明从 Tauri 角度看，Bun 是被支持的，不是临时凑数。

## 为什么我之前没有优先推荐 Bun

### 1. 初始化阶段更怕变量太多

你现在做的是桌面壳初始化，不是做前端包管理器评测。

这个阶段最重要的是：

- 项目先生成成功。
- 目录结构先固定。
- Rust 后端先跑起来。
- 前端先能显示。

Bun 当然也能做到，但它会多引入一层“我现在到底要不要把整个前端工具链也换成 Bun 生态”的决策。

### 2. Bun 的优势不一定是你当前最需要的

Bun 常见优势是：

- 启动快。
- 安装快。
- 工具链一体化。

但对 Aether_island 来说，当前更重要的是：

- 兼容性稳定。
- 团队/个人习惯统一。
- 后面写 Tauri + Rust 时少遇到奇怪的工具链问题。

所以“最快”不一定比“最稳”更重要。

### 3. 未来维护口径更简单

如果你以后让我 review 或帮你排查问题，`pnpm` / `npm` 的路径更常规，社区样例也更多。

这不代表 Bun 不行，只是：

```text
常规工具链 = 更少解释成本
```

对一个还在搭骨架的项目，这个很有价值。

## 那什么时候可以选 Bun

如果你满足下面这些条件，直接选 Bun 完全可以：

- 你本机已经装好 Bun。
- 你平时就习惯用 Bun。
- 你愿意接受少量生态边缘情况。
- 你想把前端工具链尽量统一到 Bun。

这种情况下，`bun` 是合理选项，不会让 Tauri 变重。

## 对 Aether_island 的实际建议

如果你想要的是：

```text
最稳、最常规、最少后续解释成本
```

优先选：

```text
pnpm
```

如果你想要的是：

```text
工具链更一体化、自己也习惯 Bun
```

那就选：

```text
bun
```

## 一句话口径

> Bun 不是不能用，而是我在“先把 Aether_island 跑稳”的阶段，更偏向 pnpm/npm 这种更常规的选择；如果你本地 Bun 已经成熟可用，选 Bun 也完全成立。

## 参考来源

- Tauri Create Project：https://v2.tauri.app/start/create-project/
- Tauri Develop：https://v2.tauri.app/develop/
- Tauri 1.5 Release（提到 Bun support）：https://v2.tauri.app/blog/tauri-1-5/
