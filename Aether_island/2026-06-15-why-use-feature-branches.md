# 为什么不直接在 main 上开发

## 结论

不直接在 `main` 上开发，是为了让 `main` 始终代表一个“相对稳定、可回退、可发布”的版本。

对 Aether_island 来说，当前已经有一个干净起点：

```text
chore: initialize tauri project
```

这时开功能分支：

```powershell
git switch -c feat/island-shell-foundation
```

可以让后续窗口、托盘、Win32、UI、Widget 等实验性改动都在分支里进行。即使写坏了，也不会污染 `main`。

## main 分支应该承担什么角色

`main` 不一定永远是生产环境，但至少应该是：

- 能构建。
- 能运行。
- 变更历史清楚。
- 适合作为回退点。
- 适合以后打 tag 或发布。

简单说：

```text
main = 稳定线
feature branch = 工作线
```

## 为什么 Aether_island 特别适合开分支

你这个项目接下来会碰到很多容易出错的点：

- Tauri 窗口配置。
- Rust 模块拆分。
- Win32 unsafe 调用。
- 鼠标穿透。
- 系统托盘。
- 异步调度。
- Widget 接口。
- 缓存与配置。

这些改动不一定一步就对。功能分支可以让你边试边提交，最后再合并到 `main`。

## 如果直接在 main 上写会怎样

不是不能写，而是会增加风险：

- 写到一半 `main` 可能跑不起来。
- 多个功能混在一起，难以审查。
- 想回退时不好判断该回退哪一段。
- 以后发布时不知道哪个提交是稳定点。

尤其是你现在在学习 Rust 和 Tauri，分支能给你更多试错空间。

## 推荐分支模型

当前可以很简单：

```text
main
└── feat/island-shell-foundation
```

后续按功能开分支：

```text
feat/window-click-through
feat/tray-menu
feat/widget-runtime
feat/github-widget
docs/project-architecture
release/v0.1.0
```

不需要一开始搞复杂 Git Flow。保持小分支、小提交就够了。

## 提交流程建议

每个功能分支内：

```powershell
git status
git diff
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
pnpm build
git add .
git commit -m "feat: add island shell foundation"
```

提交前要先看 diff，不要把无关文件混进去。

## 简历与面试表达

可以这样表达：

```text
项目采用 main 稳定线与 feature branch 工作线的轻量分支模型。main 保持可构建、可运行，复杂功能如 Win32 鼠标穿透、托盘和 Widget runtime 在独立分支中开发和验证，降低实验性系统代码对稳定版本的影响。
```

## 一句话总结

> main 用来保留稳定结果，功能分支用来承载试错过程。Aether_island 接下来会涉及系统级 Rust 和 Win32 调用，开分支更安全、更适合审查和预发布。
