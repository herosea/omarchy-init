# 企业微信

本文记录在 Omarchy（Arch Linux + Hyprland/Wayland）上安装企业微信的复现步骤，以及本机实测中失败的方案。

截至 2026 年 9 月，企业微信没有面向普通 Linux 桌面的官方原生客户端。可用方案是用 Deepin Wine 运行 Windows 版。

## 当前可用组合

```text
AUR 适配层：com.qq.weixin.work.deepin 5.0.0.6008~spark2-2
企业微信：腾讯官网 Windows 版 5.0.10.6015
Wine：deepin-wine10-stable 10.14deepin11-1
运行助手：spark-dwine-helper 5.8_5.3.14-1
缩放依赖：xorg-xdpyinfo
容器：~/.deepinwine/Deepin-WXWork
启动：~/.local/bin/wecom-deepin
桌面入口：~/.local/share/applications/com.qq.weixin.work.deepin.desktop
```

实测可用：登录、主界面、2 倍缩放、中文、聊天、**截图 Ctrl+V 粘贴**。

中文候选窗口过小是 Fcitx5 在 HiDPI XWayland 上按 96 DPI 绘制导致的，不是 Wine 缩放问题。适配见 `fcitx5-hidpi.md`。

已知限制：

- 邮件、微文档依赖内置 CEF，窗口经常是 0×0 或空白。
- 工作台投屏（Miracast）远程只有鼠标，画面是黑的。Wine 抓不到 Hyprland 桌面。
- 启动后 CEF 的 `WXWorkWeb.exe --type=crashpad-handler` 会空转占 CPU。启动器会定期杀掉该进程，不影响聊天和粘贴。

粘贴快捷键是 Wine 的 **Ctrl+V**，不是 Super+V。

## 安装

```bash
omarchy pkg aur add com.qq.weixin.work.deepin
omarchy pkg add xorg-xdpyinfo
```

AUR 会拉 `deepin-wine10-stable` 和 `spark-dwine-helper`。不要安装 `com.qq.weixin.work.deepin-debug`，它会和 `deepin-wine10-stable-debug` 文件冲突。

### 容器解压

包内 `files.7z` 的 MD5 与 `files.md5sum` 不一致。新版 7-Zip 还可能拒绝 Wine 的“危险链接”。本机用 `bsdtar` 解包后才能生成：

```text
~/.deepinwine/Deepin-WXWork/
```

### 升级到官网 5.0.10

AUR 封装的 `5.0.0.6008` 会被腾讯服务端拒绝。下载：

```text
https://dldir1.qq.com/wework/work_weixin/WeCom_5.0.10.6015.exe
SHA-256: d46b1cc2603c70ff9cccd85998eed0c0d61f11a3a68e050b0695111294c10c87
```

本机副本：`/home/herosea/Projects/debs/WeCom_5.0.10.6015.exe`

企业微信完全退出后：

```bash
export WINEPREFIX="$HOME/.deepinwine/Deepin-WXWork"
/opt/deepin-wine10-stable/bin/wineserver -k
deepin-wine10-stable /home/herosea/Projects/debs/WeCom_5.0.10.6015.exe /S
```

确认版本：

```bash
7z l "$WINEPREFIX/drive_c/Program Files (x86)/WXWork/WXWork.exe" \
  | grep -E 'FileVersion|ProductVersion'
```

重置容器会回到 AUR 旧版，必须再跑一次官网安装程序。

### 缩放 2 倍

```bash
APPRUN_CMD=deepin-wine10-stable \
  /opt/spark-dwine-helper/spark-dwine-helper/scale-set-helper/set-wine-scale.sh \
  --set-scale-factor 2.0 "$HOME/.deepinwine/Deepin-WXWork"
```

会写入 `scale.txt=2.0` 和 `LogPixels=192`。完全退出再启动后生效。

### Hyprland 黑色界面

二维码或主界面发黑，是 XWayland 合成问题，不是 CEF 坏了。在 `~/.config/hypr/hyprland.lua`：

```lua
o.window({
  class = "^com\\.qq\\.weixin\\.work\\.deepin$",
  title = "^企业微信$",
}, {
  tag = "-default-opacity",
  opacity = "1 1",
  opaque = true,
})

o.window({
  class = "^com\\.qq\\.weixin\\.work\\.deepin$",
  title = "^$",
  xwayland = true,
  float = true,
}, {
  opacity = "0 0",
  no_focus = true,
  no_shadow = true,
})
```

```bash
hyprctl reload
hyprctl configerrors
```

两条都要：只设不透明，无标题阴影会变成黑蒙层；只藏阴影，主窗口仍可能过暗。

### 启动器与菜单

`~/.local/bin/wecom-deepin` 调用 Deepin 官方 `run.sh`，并在运行期间杀掉空转的 crashpad-handler。

桌面入口 `Name=企业微信`，`Categories=Network;InstantMessaging;`，`StartupWMClass=com.qq.weixin.work.deepin`。不要用 `Categories=chat;`，Omarchy 菜单可能列不出来。

启动：

```bash
wecom-deepin
```

或在菜单搜「企业微信」。

## 失败尝试

| 方案 | 结果 |
| --- | --- |
| AUR 自带 `5.0.0.6008`，不升级 | 服务端拒绝，版本过低 |
| `4.1.32.6005` + deepin-wine8 | 服务端拒绝；迁完整登录容器也不行 |
| 从 `172.16.70.108` 拷 wine8 运行时和容器 | `wow64.dll` 等符号链接指向缺失的 `/opt/apps/...`；5.0.10 安装失败 |
| Arch `wine-staging 11.16` 新容器 + 官网 5.0.10 | 能登录、聊天、2 倍缩放；**截图无法粘贴**；邮件/文档 CEF `CreateWindowEx` 错误 1400；WeMail 常为 0×0 |
| Wine 虚拟桌面 | 多出一个空白「Wine 桌面」；CEF 父窗口错误仍在 |
| Wine Wayland 驱动 | WeMail 能显示但内容空白，并盖住主界面 |
| Wayland→X11 剪贴板桥、Wine 内 `OleSetClipboard`/`CF_DIB`/`CF_HDROP` | Staging 下聊天仍贴不上图 |
| 工作台 Miracast 投屏 | 远程只有鼠标，画面黑。Wine 只能抓到黑的 XWayland 根窗口 |

`wine-staging` 与上述测试容器、剪贴板桥已从本机删除，只保留 Deepin Wine 这一套。

## 其他注意

- `spark-dwine-helper` 会调 `xdpyinfo`，缺了要装 `xorg-xdpyinfo`。
- 启动脚本访问 `com.deepin.dde.TrayManager`，Omarchy 没有该服务，日志会有 D-Bus 错误，可忽略。
- 另一个 AUR 包 `com.qq.weixin.work.deepin.gitee` 仍是 `5.0.0.6008`，不能当替代版。

## 卸载

```bash
omarchy pkg drop com.qq.weixin.work.deepin
```

确认不要数据后再删：

```text
~/.deepinwine/Deepin-WXWork/
```

不要清空整个 `~/.deepinwine/`。删依赖前用 `pacman -Qi <包名>` 看 `Required By`。
