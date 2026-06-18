# Aether_island 透明窗口阶段记录

## 结论

Aether_island 已从普通小窗口推进到透明无边框浮层阶段。

## 已改动

### tauri.conf.json

窗口配置升级为：

```json
{
  "title": "Aether_island",
  "width": 480,
  "height": 120,
  "resizable": false,
  "decorations": false,
  "transparent": true,
  "alwaysOnTop": true,
  "skipTaskbar": true,
  "shadow": false,
  "backgroundColor": "#00000000"
}
```

### CSS

- `body` 设置为 `pointer-events: none`
- `.island` 设置 `pointer-events: auto`
- `.island` 增加 `cursor: default`

## 验证结果

以下命令已通过：

```powershell
pnpm build
cargo fmt --check --manifest-path ".\src-tauri\Cargo.toml"
cargo clippy --manifest-path ".\src-tauri\Cargo.toml" -- -D warnings
cargo test --manifest-path ".\src-tauri\Cargo.toml"
```

## 未验证项

需要人工运行 `pnpm tauri dev` 观察：

- 透明背景是否生效
- 是否仍有边线
- 是否真正不显示任务栏
- 是否置顶
- 胶囊 hover 是否仍顺滑

## 下一步

如果透明无边框外观满意，下一阶段就可以进入：

```text
Win32 鼠标穿透
```

这是更底层的一步，需要在窗口句柄层处理 `WS_EX_TRANSPARENT` 和 `WS_EX_LAYERED`。
