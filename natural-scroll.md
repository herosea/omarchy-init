# 鼠标与触摸板自然滚动

本文记录如何在 Omarchy 的 Hyprland 中启用与 macOS 相同的自然滚动方向：滚动时，页面内容跟随操作方向移动。

## 配置

在 `~/.config/hypr/input.lua` 中添加：

```lua
-- Match macOS-style natural scrolling: content follows finger movement.
hl.config({
  input = {
    natural_scroll = true,
    touchpad = {
      natural_scroll = true,
    },
  },
})
```

其中：

- `input.natural_scroll`：启用鼠标滚轮自然滚动。
- `input.touchpad.natural_scroll`：启用触摸板自然滚动。

## 应用与验证

Hyprland 通常会在保存配置后自动重载。也可以手动重载并检查配置错误：

```bash
hyprctl reload
hyprctl configerrors
```

`hyprctl configerrors` 无输出即表示配置解析正常。
