# Fcitx5 候选窗口过小（HiDPI / XWayland）

企业微信和飞书里，中文候选窗口只有正常界面大约一半大。原生 Wayland 应用（如 foot）一般正常。

本机已按下面配置改过，飞书、企业微信、foot 的候选窗口大小均正常。企业微信改完 Fcitx5 后若输入卡住，重启一次客户端即可。

## 原因

本机屏幕是 2880×1800，Hyprland 缩放为 2，并且开启了 `xwayland.force_zero_scaling`。XWayland 向客户端报告 96 DPI：

```text
dimensions: 2880x1800 pixels (762x476 millimeters)
resolution: 96x96 dots per inch
```

企业微信（Deepin Wine）和飞书（Electron）窗口都是 XWayland，实际输入走 XIM。Fcitx5 在 XWayland 上会忽略 RandR 屏幕 DPI，只用 `Xft.dpi`（未设置）或 96。候选窗口按 1 倍绘制，应用界面按 2 倍绘制，所以看起来很小。

Fcitx5 诊断里可以看到对应的输入上下文：

```text
program:wine-preloader frontend:xim
program:feishu frontend:xim
```

不要用全局 `Xft.dpi: 192` 来修。它会影响所有 X11 程序，叠上已有的 `GDK_SCALE=2` 和 Wine `LogPixels=192` 可能变成 4 倍。

## 配置

先停 Fcitx5，再写 `~/.config/fcitx5/conf/classicui.conf`。Fcitx5 退出时会把内存里的配置写回文件；带着旧进程改文件会被覆盖。

停掉 `omarchy-fcitx5.service` 后，D-Bus 可能再拉起一个不带 `--disable notificationitem` 的 `/usr/bin/fcitx5`。若 `systemctl --user start` 立刻退出，先结束那个多余进程再启动服务。

```ini
Font="Noto Sans CJK SC 20"
MenuFont="Noto Sans CJK SC 20"
PerScreenDPI=True
ForceWaylandDPI=48
EnableFractionalScale=True
```

```bash
systemctl --user stop omarchy-fcitx5.service
# 若仍有 /usr/bin/fcitx5，先结束它
# 此时写入 classicui.conf
systemctl --user start omarchy-fcitx5.service
```

默认字号是 10pt。X11 候选窗口没有合成器缩放，所以改成 20pt。Wayland 候选窗口仍会被合成器乘 2，因此把 `ForceWaylandDPI` 设为 48（`96 × 10 / 20`），原生 Wayland 应用的候选窗口保持原来大小。

若以后改字号，按下面关系一起改 DPI：

```text
ForceWaylandDPI = 96 × 10 / <Font 磅值>
```

本机已安装 `Noto Sans CJK SC`。

## 验证

在企业微信或飞书的输入框按 `Ctrl + Space` 切到雾凇拼音，打几个字。候选窗口应和应用里的中文字号接近，且不挡字。

在 foot 里同样试一次，确认 Wayland 候选窗口没有明显变大。

本机实测：飞书、企业微信、foot 均正常。

## 备份

无。此前没有 `classicui.conf`，用的是 Fcitx5 默认值。
