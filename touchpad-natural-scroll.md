# 触摸板自然滚动

本文记录如何在 Omarchy 的 Hyprland 中启用与 macOS 相同方向的触摸板自然滚动：双指向上移动时，页面内容也随手指向上移动。

## 配置

在 `~/.config/hypr/input.lua` 中添加：

```lua
-- Match macOS-style touchpad scrolling: content follows finger movement.
hl.config({
  input = {
    touchpad = {
      natural_scroll = true,
    },
  },
})
```

该配置只影响触摸板滚动方向，不改变鼠标滚轮方向、触摸板灵敏度或手势设置。

## 应用与验证

Hyprland 通常会在保存配置后自动重载。也可以手动重载并检查配置错误：

```bash
hyprctl reload
hyprctl configerrors
```

`hyprctl configerrors` 无输出即表示配置解析正常。

