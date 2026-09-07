# Omarchy 双显示器布局

本文记录本机笔记本屏幕与外接显示器的 Hyprland 配置。

## 显示器

| 输出 | 分辨率 | 缩放 | 逻辑尺寸 | 位置 |
|---|---:|---:|---:|---:|
| `eDP-1`（笔记本） | 2880×1800 | 1.6 | 1800×1125 | `0x330` |
| `HDMI-A-1`（外接 Dell） | 3840×2160 | 2 | 1920×1080 | `1800x0` |

笔记本位于左侧，外接显示器位于右侧。外接显示器的底边对应笔记本屏幕从顶部算约 `2/3` 的高度，并从该位置向上延伸。

垂直位置计算：

```text
笔记本逻辑高度 = 1800 / 1.6 = 1125
笔记本的 2/3 高度 = 1125 × 2/3 = 750
外接屏逻辑高度 = 2160 / 2 = 1080
垂直偏移 = 1080 - 750 = 330
```

因此将外接屏放在 `y=0`，笔记本放在 `y=330`。笔记本逻辑宽度为 `1800`，所以右侧外接屏从 `x=1800` 开始。

## 配置

编辑 `~/.config/hypr/monitors.lua`：

```lua
local omarchy_gdk_scale = 2

hl.env("GDK_SCALE", tostring(omarchy_gdk_scale))

-- Laptop on the left: 2880x1800 / 1.6 = 1800x1125 logical pixels.
-- Its top is 330px below the external display, placing the external display's
-- bottom edge at two-thirds of the laptop display's height.
hl.monitor({ output = "eDP-1", mode = "preferred", position = "0x330", scale = 1.6 })

-- External display on the right: 3840x2160 / 2 = 1920x1080 logical pixels.
hl.monitor({ output = "HDMI-A-1", mode = "preferred", position = "1800x0", scale = 2 })
```

`mode = "preferred"` 当前使外接屏运行在 3840×2160@60Hz，笔记本屏运行在 2880×1800@120Hz。

## 应用与验证

Hyprland 通常会在保存后自动重载，也可以手动执行：

```bash
hyprctl reload
hyprctl configerrors
hyprctl monitors
```

`hyprctl configerrors` 无输出表示配置解析正常。

## 备份

修改前的配置保存在：

```text
~/.config/hypr/monitors.lua.bak.20260907-170711
```
