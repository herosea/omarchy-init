# Fcitx5 + 雾凇拼音

本文记录在 Omarchy 上安装和配置 Fcitx5、Rime 与雾凇拼音（Rime Ice，全拼），并使用 `Ctrl + Space` 全局切换中英文输入。

## 安装依赖

```bash
sudo pacman -S --needed fcitx5 fcitx5-gtk fcitx5-qt fcitx5-rime git
```

Omarchy 已提供并启用用户服务 `omarchy-fcitx5.service`，不要再通过 Hyprland 或 XDG autostart 启动第二个 Fcitx5 进程，否则两个实例会争用 D-Bus 名称。

## 安装雾凇拼音

从官方仓库浅克隆最新配置，避免 `rime-ice-pinyin-git` AUR 包下载完整 Git 历史：

```bash
git clone --depth 1 https://github.com/iDvel/rime-ice.git /tmp/omarchy-rime-ice
install -d ~/.local/share/fcitx5/rime
```

按照上游 `recipe.yaml` 的 `install_files` 列表，将以下内容复制到 `~/.local/share/fcitx5/rime/`：

```text
cn_dicts/
en_dicts/
opencc/
lua/
default.yaml
squirrel.yaml
weasel.yaml
rime_ice.schema.yaml
rime_ice.dict.yaml
t9.schema.yaml
double_pinyin*.schema.yaml
symbols_v.yaml
symbols_caps_v.yaml
radical_pinyin.schema.yaml
radical_pinyin.dict.yaml
melt_eng.schema.yaml
melt_eng.dict.yaml
custom_phrase.txt
```

部署 Rime 数据：

```bash
rime_deployer --build \
  ~/.local/share/fcitx5/rime \
  /usr/share/rime-data \
  ~/.local/share/fcitx5/rime/build
```

部署成功后应存在：

```text
~/.local/share/fcitx5/rime/build/rime_ice.schema.yaml
~/.local/share/fcitx5/rime/build/rime_ice.table.bin
```

## 配置 Fcitx5

写入 `~/.config/fcitx5/config`：

```ini
[Hotkey]
TriggerKeys=
0=Control+space
AltTriggerKeys=
0=Shift_L
ModifierOnlyKeyTimeout=500

[Behavior]
ActiveByDefault=True
ShareInputState=All
```

写入 `~/.config/fcitx5/profile`：

```ini
[Groups/0]
# Group Name
Name=Default
# Layout
Default Layout=us
# Default Input Method
DefaultIM=rime

[Groups/0/Items/0]
# Name
Name=keyboard-us
# Layout
Layout=

[Groups/0/Items/1]
# Name
Name=rime
# Layout
Layout=

[GroupOrder]
0=Default
```

必须先停止 Fcitx5，再写入 `profile`，最后重新启动服务。Fcitx5 会在退出时保存内存中的旧配置；如果先改文件再重启，旧配置可能覆盖刚写入的 Rime 配置。

```bash
systemctl --user stop omarchy-fcitx5.service
# 此时写入 ~/.config/fcitx5/config 和 ~/.config/fcitx5/profile
systemctl --user start omarchy-fcitx5.service
```

## 配置全局中英文切换

部分 Wayland 应用不会可靠地把 `Ctrl + Space` 交给 Fcitx5。为了在所有应用中稳定切换，在 `~/.config/hypr/bindings.lua` 添加 Hyprland 全局快捷键：

```lua
-- Toggle between the US keyboard and Rime (Wusong Pinyin) globally.
o.bind("CTRL + SPACE", "Toggle Chinese input", "fcitx5-remote -t")
```

应用并检查 Hyprland 配置：

```bash
hyprctl reload
hyprctl configerrors
```

`hyprctl configerrors` 应无输出。

## 验证

```bash
systemctl --user is-active omarchy-fcitx5.service
fcitx5-remote
fcitx5-remote -n
fcitx5-remote -m rime
```

预期结果：

- 服务状态为 `active`；
- 中文状态下 `fcitx5-remote` 输出 `2`；
- 当前输入法为 `rime`；
- `fcitx5-remote -m rime` 输出 `rime`；
- 按 `Ctrl + Space` 可在 `rime` 与 `keyboard-us` 之间切换，此方式已验证可用。

## 已知问题

虽然已按照 Fcitx5 的 modifier-only 机制将左 `Shift` 配置为 `AltTriggerKeys`，并将 `ModifierOnlyKeyTimeout` 设置为 500ms，但在当前 Omarchy + Hyprland + Fcitx5 Wayland 环境中，单独轻按左 `Shift` 仍不能切换中英文。

目前统一使用已验证可用的 `Ctrl + Space` 全局快捷键；不要依赖左 `Shift` 切换。后续查明 Wayland 修饰键释放事件的兼容性问题后再更新此配置。

## 备份

本次配置过程中创建了以下备份：

```text
~/.config/fcitx5/profile.bak-before-wusong
~/.config/hypr/bindings.lua.bak-before-fcitx-toggle
```
