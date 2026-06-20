# Hover Jitter Native Style Refresh Fix

日期：2026-06-19

## 问题

用户反馈：上一轮稳定 hitbox 后，鼠标滑过灵动岛仍然偶发抽搐，但比较难触发。

## 进一步判断

第一轮修复解决的是视觉层缩放导致的 hover hitbox 不稳定。但偶发抽搐仍存在，说明还有第二层因素。

本轮读代码后发现：

1. 前端每次 `mouseenter` / `mouseleave` 都会调用 `set_shell_state`。
2. Rust 收到 `set_shell_state` 后会调用 `apply_click_through_for_state`。
3. `apply_click_through_for_state` 会调用 Win32 `SetWindowLongPtrW` / `SetWindowPos` 刷新窗口扩展样式。
4. 当前默认策略 `InteractiveOnly` 下，hovered 和 collapsed 实际都对应 `ClickThroughMode::Disabled`。
5. 也就是说，即使模式没有变化，窗口仍可能重复做原生样式刷新。

这类 Win32 样式刷新发生在鼠标 hover 边界附近时，可能造成偶发输入命中/窗口刷新抖动。

## 修改

### 1. 前端状态去重

文件：

```text
D:\Aether_island\src\main.ts
```

新增 `currentIslandState`，如果目标状态和当前状态相同，就不再重复：

- 切换 class。
- 调用 Tauri IPC。
- 通知 Rust shell state。

同时去掉启动时对 collapsed 的重复设置，因为 HTML 初始已经是 `island--collapsed`。

### 2. Rust click-through mode 缓存

文件：

```text
D:\Aether_island\src-tauri\src\lib.rs
```

新增：

```rust
click_through_mode: Mutex<Option<ClickThroughMode>>
```

使用 `Option` 的原因：

- `None` 表示还没有向 Win32 应用过模式。
- 第一次一定要应用，确保 `WS_EX_LAYERED` 被补上。
- 后续如果模式没变，就跳过 Win32 调用。

本轮中间曾把初始值设成 `Some(Disabled)` 的等价逻辑，结果烟测发现主窗口缺 `Layered=true`。这说明初始化缓存不能假装已经应用过 Win32 样式。

### 3. Win32 层样式无变化保护

文件：

```text
D:\Aether_island\src-tauri\src\window_utils.rs
```

如果计算出的 extended style 和当前 style 完全相同，直接返回，不再调用：

- `SetWindowLongPtrW`
- `SetWindowPos`

这是最后一层保险。

## 小白解释

之前像是：鼠标每碰一下岛，哪怕状态其实没变，程序都要去 Windows 那边重新说一遍“窗口样式帮我刷新一下”。

这种刷新大多数时候没事，但刚好发生在鼠标边缘、动画边缘，就可能让 hover 判断抖一下。

现在改成：

```text
状态真的变了 -> 才通知 Rust
窗口模式真的变了 -> 才调 Win32
Win32 样式真的变了 -> 才刷新窗口
```

这样从三层都减少无意义抖动源。

## 验证

在 `D:\Aether_island` 运行：

```powershell
pnpm check
pnpm smoke:window
```

结果：

- `pnpm check` 通过。
- Rust 18 个测试通过。
- Clippy 通过。
- 前端 build 通过。
- Python 7 个测试通过。

第一次烟测暴露出初始化缓存错误：

```text
Layered=false
```

修正为 `Option<ClickThroughMode>` 后重新运行：

- `pnpm smoke:window` 通过。
- 主窗口恢复 `Layered=true`。
- 窗口断言通过。

## 剩余风险

这个修复减少了重复 IPC 和重复 Win32 样式刷新，理论上会降低偶发抽搐。如果用户仍能复现，下一步需要做更强的 hover hysteresis：

- 外层命中盒扩大到比视觉胶囊更高。
- `mouseleave` 时读取鼠标坐标，只有离开安全区域才 collapse。
- 或者暂时禁用 hover 时的 Win32 click-through 相关同步，只保留视觉 hover。
