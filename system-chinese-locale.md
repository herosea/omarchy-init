# 系统中文语言环境

本文记录在 Omarchy（Arch Linux）上启用简体中文系统语言。中文字体与中文输入法是独立功能；字体和输入法请分别安装、配置，输入法配置见 `fcitx5-rime-ice.md`。

## 安装中文字体

```bash
sudo pacman -S --needed noto-fonts-cjk noto-fonts-emoji
```

## 启用中文 Locale

取消 `/etc/locale.gen` 中下面一行的注释：

```text
zh_CN.UTF-8 UTF-8
```

生成 locale：

```bash
sudo locale-gen
```

将系统默认语言设为简体中文：

```bash
sudo localectl set-locale LANG=zh_CN.UTF-8
```

等价的 `/etc/locale.conf` 内容为：

```ini
LANG=zh_CN.UTF-8
```

不要设置全局 `LC_ALL`。保留各类 `LC_*` 未设置时，它们会继承 `LANG`，应用也仍可按需覆盖单项格式。

## 应用配置

注销 Omarchy 会话并重新登录；若个别长期运行的服务仍显示英文，重启系统即可。语言设置不会完整地热更新到已经运行的桌面进程。

## 验证

重新登录后运行：

```bash
locale
localectl status
```

预期 `LANG` 为 `zh_CN.UTF-8`。没有提供中文翻译的应用仍会显示英文，这是应用自身翻译资源的限制。
