# 企业微信

本文记录在 Omarchy（Arch Linux + Hyprland/Wayland）上安装企业微信的调研和实测结果。

截至 2026 年 9 月，企业微信没有面向普通 Linux 桌面的官方原生客户端。Arch Linux 上现有方案主要通过 Deepin Wine 运行 Windows 版企业微信，兼容性取决于企业微信版本、Wine 版本和随应用提供的 DLL，不能仅凭“可以启动”判断是否可用。

## 推荐结论

暂不安装 AUR 当前提供的 `5.0.0.6008~spark2-2`。

该版本在本机可以启动并生成登录二维码，但 CEF/WebView 将二维码绘制为深灰色码和黑色背景，无法正常扫码。强制 Wine 使用浅色主题后问题仍然存在；登录后的工作台、微盘、微文档和第三方应用也依赖相同的 WebView，因此不适合作为日常办公客户端。

更值得尝试的旧版组合是：

```text
企业微信：4.1.32.6005
Wine：deepin-wine8-stable
DLL：保留 4.1.32.6005 软件包随附的 DLL
```

目前星火镜像已经删除这个旧版，统信官方仓库对应地址只返回零字节占位文件，Internet Archive 也没有安装包存档。在取得来源可信、哈希可验证的旧版 `.deb` 之前，不要从不明软件下载站安装。

历史包名为：

```text
com.qq.weixin.work.deepin_4.1.32.6005deepin11~spark1_all.deb
```

## AUR 软件包说明

当前主要软件包：

```text
com.qq.weixin.work.deepin
```

安装入口为：

```bash
omarchy pkg aur add com.qq.weixin.work.deepin
```

另一个 AUR 包 `com.qq.weixin.work.deepin.gitee` 不是更稳定的企业微信版本。它同样封装 `5.0.0.6008~spark2-2`，企业微信主程序仍来自山东大学星火镜像；“gitee”主要指部分附加资源改从 Gitee 下载，不能解决 WebView 问题。

## 5.0 版实测记录

本机曾安装以下版本进行验证，验证完成后已全部卸载和清理：

```text
com.qq.weixin.work.deepin 5.0.0.6008~spark2-2
deepin-wine10-stable 10.14deepin11-1
spark-dwine-helper 5.8_5.3.14-1
```

### 调试包文件冲突

AUR 构建会同时生成主程序包和调试包：

```text
com.qq.weixin.work.deepin
com.qq.weixin.work.deepin-debug
```

`com.qq.weixin.work.deepin-debug` 与 `deepin-wine10-stable-debug` 包含相同 build-id 的调试文件，安装时会报文件冲突。调试包不是程序运行依赖，不要使用 `--overwrite` 覆盖文件；若只是测试主程序，可以只安装主包：

```bash
sudo pacman -U --needed \
  ~/.cache/yay/com.qq.weixin.work.deepin/com.qq.weixin.work.deepin-5.0.0.6008~spark2-2-x86_64.pkg.tar.zst
```

### Wine 容器解压失败

当前包中的 `files.7z` 实际 MD5 与 `files.md5sum` 不一致。首次启动还可能因新版 7-Zip 的解包行为而无法创建：

```text
~/.deepinwine/Deepin-WXWork/
```

本机使用 `bsdtar` 手动解包后可以完成容器初始化，但这只能解决启动问题，不能修复 CEF/WebView。

### 缺少缩放检测依赖

`spark-dwine-helper` 的首次启动脚本调用 `xdpyinfo`，但 AUR 依赖未覆盖该命令。缺少时日志会显示：

```text
get-scale.sh: xdpyinfo: command not found
```

对应 Arch 软件包为：

```bash
omarchy pkg add xorg-xdpyinfo
```

### Deepin 托盘错误

启动脚本会访问 Deepin 桌面的 D-Bus 服务：

```text
com.deepin.dde.TrayManager
```

Omarchy 不提供该服务，因此会出现 `org.freedesktop.DBus.Error.ServiceUnknown`。本机实测该错误会产生大量告警，但不是二维码显示异常的直接原因。

### WebView 故障

本机最终状态为：

- `WXWork.exe` 进程持续运行；
- Hyprland 能识别并映射 `WeCom` 窗口；
- 登录二维码已经生成；
- 二维码和背景均接近黑色，无法正常扫码；
- 设置 `AppsUseLightTheme=1` 和 `SystemUsesLightTheme=1` 后无改善。

这说明故障位于该版本的 CEF/DLL/Wine 绘制兼容层，而非网络、普通深色主题或窗口未启动。

## 安装前检查

如果以后重新取得 `4.1.32.6005` 安装包，应先完成以下检查：

1. 确认来源是腾讯、Deepin、统信或星火的历史官方文件，而不是软件下载站重新封装的文件。
2. 对照 AUR 历史提交中的 SHA-256 校验值验证文件。
3. 解包检查 `files.7z`、`files.md5sum`、`files/dlls/` 和启动脚本是否完整。
4. 在不覆盖现有文件的情况下构建独立 Arch 软件包。
5. 依次测试登录二维码、中文输入、聊天、文件传输、工作台、微盘、微文档和第三方 WebView 应用。
6. 验证通过后再考虑锁定版本，避免被 AUR 自动升级到不兼容的 5.0 包。

## 卸载与还原

卸载主程序：

```bash
omarchy pkg drop com.qq.weixin.work.deepin
```

确认不再需要其中的数据后，删除 Wine 容器：

```text
~/.deepinwine/Deepin-WXWork/
```

不要直接清空整个 `~/.deepinwine/`，其中可能包含其他 Deepin Wine 应用的数据。

若安装过程中生成了 AUR 构建缓存，可按具体包名清理：

```text
~/.cache/yay/com.qq.weixin.work.deepin/
~/.cache/yay/deepin-wine8-stable/
~/.cache/yay/deepin-wine10-stable/
~/.cache/yay/spark-dwine-helper/
```

删除依赖前应先用 `pacman -Qi <包名>` 检查 `Required By`，不要使用宽泛的递归删除命令，以免移除其他软件正在使用的组件。
