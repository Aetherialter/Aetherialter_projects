# Tauri 当前目录初始化注意事项

## 结论

如果已经位于目标项目目录：

```text
D:\Aether_island
```

在 `create-tauri-app` 的项目目录提示中不应该再输入：

```text
Aether_island
```

否则很可能生成嵌套目录：

```text
D:\Aether_island\Aether_island
```

更合理的方式是使用当前目录：

```text
.
```

## 当前场景

用户当前已经执行：

```powershell
Set-Location -LiteralPath "D:\Aether_island"
irm https://create.tauri.app/ps | iex
```

脚手架提示：

```text
? Project name (tauri-app) ›
```

此时如果目标是“就在 `D:\Aether_island` 这个文件夹初始化项目”，应该输入：

```text
.
```

## 原因

很多项目脚手架中的 `Project name` 实际承担两个角色：

- 新项目目录名。
- 初始包名或应用名的来源。

当命令不是显式传入当前目录时，输入项目名通常会创建同名子目录。

因此：

```text
当前目录 = D:\Aether_island
输入 Aether_island
结果可能 = D:\Aether_island\Aether_island
```

## 推荐处理

如果还停在交互提示处：

```text
? Project name (tauri-app) ›
```

直接输入：

```text
.
```

如果脚手架后续询问 package name 或 app name，再输入：

```text
Aether_island
```

如果脚手架不接受 `.`，应取消当前流程，再使用显式当前目录命令：

```powershell
npm create tauri-app@latest .
```

## 工程规则

初始化项目时要区分：

- 文件夹路径。
- 项目显示名。
- package name。
- Tauri identifier。

对 Aether_island 来说推荐：

```text
目录路径：D:\Aether_island
项目名：Aether_island
package name：aether-island
identifier：com.aetherialter.aether-island
```

## 参考来源

- Tauri 官方创建项目文档：https://v2.tauri.app/start/create-project/
- create-tauri-app 官方仓库：https://github.com/tauri-apps/create-tauri-app
