# Python Daemon Protocol Sketch

日期：2026-06-19

## 结论

本轮把 `island-daemon` 定位为 v0.4 阶段的 Python sidecar 协议草图，而不是正式双进程运行时。Rust 主体仍然负责窗口、Widget Runtime 和前端事件链；Python 侧先验证 stdio JSON 协议、消息解析、生命周期退出语义。

这符合当前阶段目标：先把 Rust-first 主体框架打牢，再给后续 Python 小组件或数据插件留下可测试的接入口。

## 当前边界

`D:\Aether_island\island-daemon` 当前提供：

- `DAEMON_READY`：daemon 启动后声明已就绪。
- `SHELL_STATE`：接收 Rust shell 状态，例如 `hovered`、`collapsed`。
- `RENDER_UPDATE`：把 Python 侧状态转换成可被岛渲染的数据。
- `SHUTDOWN`：接收关闭消息并正常退出。

Rust 当前还没有自动启动 Python 进程。这个选择是刻意的：自动拉起 sidecar 会立刻引入子进程生命周期、退出清理、崩溃重启、打包路径、日志收集等复杂度。v0.4 先把协议和 Rust 侧 `SidecarWidget` 边界留下来，更利于小步验证。

## 本轮修复

### 1. TextIO 导入来源

`TextIO` 改为从 `typing` 导入。

原因：当前验证环境 Python 3.14 下，`collections.abc.TextIO` 不可用，导致测试导入阶段失败。`typing.TextIO` 更适合这里表达“类文件对象”的静态类型意图。

### 2. Ruff import 排序

`tests/test_daemon.py` 调整为标准库 import 分组和排序。

原因：项目启用了 ruff 的 `I` 规则，import 顺序属于质量门禁的一部分。

### 3. Python 生成物忽略

新增 `island-daemon\.gitignore`，忽略：

- `.venv/`
- `.pytest_cache/`
- `.ruff_cache/`
- `__pycache__/`
- `*.py[cod]`

原因：这些都是本地环境或工具缓存，不应该进入项目历史。sidecar 是协议代码，虚拟环境不是源码。

## 验证结果

在 `D:\Aether_island\island-daemon` 运行：

```powershell
uv run ruff format .
uv run ruff check .
uv run pytest
```

结果：

- ruff format：通过，5 个文件保持不变。
- ruff check：通过。
- pytest：通过，6 个测试全部通过。

在 `D:\Aether_island` 运行：

```powershell
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
pnpm build
```

结果：

- Rust 格式检查通过。
- Rust clippy 通过。
- Rust test 通过，目前没有 Rust 单元测试用例。
- 前端 production build 通过。

## 对 v0.4 的意义

v0.4 的主线不应变成“立刻完成重型双进程架构”，而应是：

1. Rust 主体稳定运行。
2. Widget Runtime 有清晰接口。
3. 前端只消费 snapshot，不绑定具体数据来源。
4. Python sidecar 协议可以独立测试。
5. 未来 Rust 拉起 Python 时，有现成的消息契约可以接入。

## 下一步建议

下一步可以补 Rust 侧 runtime/provider 的单元测试，尤其是 active widget 选择、优先级 fallback、错误保留和 shutdown 调用。这样在继续加小组件之前，先把核心调度逻辑钉住。
