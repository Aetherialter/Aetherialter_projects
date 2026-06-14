# Aether_island 初始 Tauri 入口链路分析

## 结论

当前项目是标准的 Tauri v2 + Vite + Vanilla TypeScript 结构。

启动链路可以简化成：

```text
pnpm tauri dev
↓
Tauri CLI 启动 Vite dev server
↓
Cargo 编译并运行 Rust 后端
↓
src-tauri/src/main.rs 调用 aether_island_lib::run()
↓
src-tauri/src/lib.rs 创建 Tauri Builder
↓
Tauri 加载 tauri.conf.json
↓
创建桌面窗口并打开 http://localhost:1420
↓
Vite 加载 index.html
↓
index.html 加载 src/main.ts
↓
前端通过 invoke("greet") 调用 Rust greet 命令
```

当前代码还只是模板状态，没有真正进入 Aether_island 的灵动岛业务。

## 文件职责

## `src-tauri/src/main.rs`

当前内容：

```rust
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

fn main() {
    aether_island_lib::run()
}
```

这个文件是 Rust 二进制程序入口。

### 关键点 1：`main()` 是真正入口

Rust 程序从 `main()` 开始执行。这里它没有直接构建 Tauri，而是调用：

```rust
aether_island_lib::run()
```

这说明项目把主要应用逻辑放在 library crate 里，也就是 `lib.rs`。

### 关键点 2：为什么有 `windows_subsystem`

这一行：

```rust
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]
```

意思是：

```text
如果不是 debug 构建，就把 Windows 子系统设置为 windows
```

效果是 release 版本启动时不会额外弹出控制台窗口。

这对桌面应用很重要。否则用户打开 Aether_island 时可能会同时出现一个黑色命令行窗口。

## `src-tauri/src/lib.rs`

当前核心内容：

```rust
#[tauri::command]
fn greet(name: &str) -> String {
    format!("Hello, {}! You've been greeted from Rust!", name)
}

pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_opener::init())
        .invoke_handler(tauri::generate_handler![greet])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

这个文件是 Tauri 应用的主要构建入口。

### `#[tauri::command]`

`#[tauri::command]` 把普通 Rust 函数暴露给前端调用。

前端可以通过：

```ts
invoke("greet", { name: "..." })
```

调用 Rust 里的：

```rust
fn greet(name: &str) -> String
```

这就是 Tauri 的前后端通信方式之一。

### `tauri::Builder::default()`

这是 Tauri 应用构建器。

你可以把它理解成：

```text
我要创建一个 Tauri 应用
接下来我要给它注册插件、命令、窗口配置、生命周期钩子
```

### `.plugin(tauri_plugin_opener::init())`

这是注册打开外部链接或文件的插件。

模板默认带上它，方便示例页面打开链接。

### `.invoke_handler(...)`

这一行注册前端可调用的 Rust 命令：

```rust
.invoke_handler(tauri::generate_handler![greet])
```

后续 Aether_island 里可能会注册这些命令：

- 获取缓存快照。
- 切换窗口穿透。
- 更新 shell 状态。
- 读取配置。
- 触发 widget 刷新。

### `.run(tauri::generate_context!())`

这里真正启动 Tauri 应用。

`generate_context!()` 会读取 Tauri 配置、资源、权限等编译期上下文。

### 当前的 `expect`

模板里用了：

```rust
.expect("error while running tauri application");
```

这在模板阶段可以接受。后续如果要做更严肃的错误处理，可以让启动错误更可诊断。

## `src/main.ts`

当前内容是前端示例逻辑。

它做了三件事：

1. 从 Tauri API 导入 `invoke`。
2. 找到页面上的 input 和 message 元素。
3. 表单提交时调用 Rust 的 `greet` 命令。

关键代码：

```ts
greetMsgEl.textContent = await invoke("greet", {
  name: greetInputEl.value,
});
```

这说明：

```text
前端不是直接运行 Rust
而是通过 Tauri invoke 发送一次命令调用
Rust 执行后返回结果
前端再更新 DOM
```

后续 Aether_island 的 UI 应该逐步从模板表单，替换成灵动岛状态渲染。

## `src/styles.css`

当前 CSS 是模板页面样式：

- logo hover 光效。
- 页面居中布局。
- input/button 默认样式。
- 深色模式适配。

这些不是 Aether_island 最终样式。后续会被替换成：

- 顶部胶囊窗口。
- 深色半透明背景。
- 弹性展开动画。
- collapsed / hovered / expanded / loading / error 状态类。

## `src-tauri/tauri.conf.json`

当前关键配置：

```json
{
  "productName": "aether-island",
  "version": "0.1.0",
  "identifier": "com.aether-island.app",
  "build": {
    "beforeDevCommand": "pnpm dev",
    "devUrl": "http://localhost:1420",
    "beforeBuildCommand": "pnpm build",
    "frontendDist": "../dist"
  },
  "app": {
    "withGlobalTauri": true,
    "windows": [
      {
        "title": "aether-island",
        "width": 800,
        "height": 600
      }
    ]
  }
}
```

### `build.beforeDevCommand`

开发时先运行：

```powershell
pnpm dev
```

也就是启动 Vite。

### `build.devUrl`

Tauri 开发窗口加载：

```text
http://localhost:1420
```

所以 `pnpm tauri dev` 本质上是：

```text
Vite 前端 dev server + Rust Tauri 桌面壳
```

### `build.frontendDist`

打包时加载：

```text
../dist
```

也就是 `pnpm build` 生成的前端静态资源。

### `app.windows`

当前窗口还是普通窗口：

```json
{
  "title": "aether-island",
  "width": 800,
  "height": 600
}
```

后续灵动岛会改成：

- 无边框。
- 小尺寸。
- 顶部居中。
- 不显示任务栏。
- 可透明。
- 可通过 Win32 设置鼠标穿透。

## Rust 知识点

## 二进制 crate 和 library crate

当前项目实际有两个 Rust 入口概念：

```text
main.rs = binary crate 入口
lib.rs = library crate 逻辑
```

`main.rs` 很薄，只负责调用 `lib.rs` 的 `run()`。

这样做的好处是：

- 应用构建逻辑更容易测试。
- 移动端入口可以复用 `run()`。
- Tauri 模板更容易兼容多平台。

## 借用：`name: &str`

`greet` 的参数是：

```rust
fn greet(name: &str) -> String
```

这里 `&str` 是字符串切片引用。Rust 没有拿走前端传入字符串的所有权，而是借用一段字符串视图来格式化输出。

返回值是：

```rust
String
```

这是新创建的拥有所有权的字符串，可以安全返回给 Tauri，再传给前端。

## `format!` 会分配新字符串

```rust
format!("Hello, {}!", name)
```

会创建一个新的 `String`。这个 `String` 的所有权由函数返回。

这和 Python 不同。Python 字符串对象由运行时管理；Rust 这里是编译期清楚地知道返回值由调用方接管。

## `expect` 的边界

模板里：

```rust
.expect("error while running tauri application");
```

如果 Tauri 启动失败，程序会 panic 并终止。

模板阶段可以接受，但后续生产化时，启动阶段错误最好能进入日志或诊断通道。

## 当前项目还没做的事

目前还没有：

- 灵动岛 UI。
- 窗口无边框。
- 鼠标穿透。
- 系统托盘。
- Widget runtime。
- 缓存。
- 配置。
- Sidecar 协议。

所以当前最重要的是先理解模板，再小步替换。

## 下一步建议

第一轮可以只做非常小的修改：

1. 修改 `tauri.conf.json` 里的窗口标题和尺寸。
2. 替换 `index.html` 与 `src/main.ts` 中的模板 greet 页面。
3. 将 `src/styles.css` 改成基础胶囊样式。
4. 保留 Rust `greet` 示例或删除它，但删除时要同步移除前端调用和 `invoke_handler`。

不建议第一轮就做 Win32 鼠标穿透。因为那会同时引入：

- raw window handle。
- windows-sys。
- unsafe。
- 平台差异。
- 鼠标交互状态机。

先把 UI 胶囊和 Tauri 启动链路跑通更稳。

## 参考来源

- Tauri Commands：https://v2.tauri.app/develop/calling-rust/
- Tauri Configuration：https://v2.tauri.app/reference/config/
- Tauri Architecture：https://v2.tauri.app/concept/architecture/
