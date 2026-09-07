# Incus 容器环境

本文记录在 Omarchy（Arch Linux）上安装和配置 Incus，并使用 `images:rockylinux/9` 创建 Rocky Linux 9 容器的过程。

## 安装

安装 Incus：

```bash
sudo pacman -S incus
```

安装后将当前用户加入具有完整管理权限的 `incus-admin` 组：

```bash
sudo usermod -aG incus-admin "$USER"
```

注销并重新登录，使组成员关系生效。可用以下命令确认：

```bash
id
getent group incus-admin
```

输出中应显示当前用户属于 `incus-admin` 组。

启用并立即启动 Incus 服务：

```bash
sudo systemctl enable --now incus.service
```

## 配置 UID/GID 映射

非特权容器需要由 Incus 守护进程使用 subordinate UID/GID。确保 `/etc/subuid` 和 `/etc/subgid` 中均有 `root` 的映射：

```text
root:1000000:1000000000
```

如果尚不存在，可执行：

```bash
grep -q '^root:' /etc/subuid || printf 'root:1000000:1000000000\n' | sudo tee -a /etc/subuid
grep -q '^root:' /etc/subgid || printf 'root:1000000:1000000000\n' | sudo tee -a /etc/subgid
sudo systemctl restart incus
```

当前用户由系统分配的映射也应保留。本机最终配置为：

```text
# /etc/subuid
herosea:100000:65536
root:1000000:1000000000

# /etc/subgid
herosea:100000:65536
root:1000000:1000000000
```

缺少 `root` 映射时，创建容器会报错：

```text
System doesn't have a functional idmap setup
```

## 初始化 Incus

运行交互式初始化：

```bash
incus admin init
```

本机采用以下布局：

- 不启用集群；
- 存储池名称为 `default`；
- 存储驱动为 `btrfs`；
- 创建托管网桥 `incusbr0`；
- IPv4 网段为 `192.168.88.0/24`，网关为 `192.168.88.1`；
- IPv4 NAT 已启用；
- IPv6 使用 Incus 自动生成的私有网段，并启用 NAT。

可查看初始化结果：

```bash
incus storage show default
incus network show incusbr0
incus profile show default
```

本机 `incusbr0` 的关键配置为：

```yaml
config:
  ipv4.address: 192.168.88.1/24
  ipv4.nat: "true"
  ipv6.address: fd42:2ddf:8b0b:cfc5::1/64
  ipv6.nat: "true"
type: bridge
managed: true
```

其中 IPv6 前缀可以由 Incus 自动生成，不必与本机完全相同。

## 确保默认 Profile 包含根磁盘

默认 profile 必须同时包含网卡和根磁盘：

```yaml
devices:
  eth0:
    name: eth0
    network: incusbr0
    type: nic
  root:
    path: /
    pool: default
    type: disk
```

如果 `incus profile show default` 中没有 `root`，执行：

```bash
incus profile device add default root disk path=/ pool=default
```

否则创建实例时会报错：

```text
Failed detecting root disk device: No root device could be found
```

## 配置 UFW

Omarchy 上启用 UFW 后，其默认入站和转发策略会阻止容器的 DHCP 和转发流量。放行 Incus 托管网桥：

```bash
sudo ufw allow in on incusbr0
sudo ufw route allow in on incusbr0
sudo ufw route allow out on incusbr0
```

这些规则同时写入 IPv4 和 IPv6 配置。可验证：

```bash
sudo ufw status verbose
```

如果未放行，容器可能只能通过 Router Advertisement 获得 IPv6 地址；IPv4 DHCP 客户端会持续发送 `DHCPDISCOVER`，但收不到 `DHCPOFFER`。

## 创建 Rocky Linux 9 容器

启动容器：

```bash
incus launch images:rockylinux/9 lab
```

查看实例及地址：

```bash
incus list
```

如果容器是在添加 UFW 规则之前启动的，可重新激活连接：

```bash
incus exec lab -- nmcli connection up "System eth0"
```

本机容器获得的 IPv4 地址为 `192.168.88.132`。DHCP 地址可能变化，不应依赖这个具体值。

验证路由和外网连通性：

```bash
incus exec lab -- ip -4 route
incus exec lab -- ping -4 -c 2 1.1.1.1
```

进入容器：

```bash
incus shell lab
```

停止、启动及删除容器：

```bash
incus stop lab
incus start lab
incus delete lab       # 需要先停止
incus delete -f lab    # 强制停止并删除
```

## 向容器转发宿主机 SSH Agent

为了让容器中的 Git 使用宿主机已经解锁的 SSH 密钥，应转发 SSH Agent，而不是把私钥复制进容器。以下示例使用名为 `debian` 的容器。

先确认宿主机 Agent socket 存在，并且已经加载所需密钥：

```bash
printf '%s\n' "$SSH_AUTH_SOCK"
test -S "$SSH_AUTH_SOCK"
ssh-add -l
```

本机使用 systemd 用户级 SSH Agent，socket 为 `/run/user/1000/ssh-agent.socket`。把它通过 Incus `proxy` 设备转发到容器：

```bash
incus config device add debian ssh-agent proxy \
  connect="unix:${SSH_AUTH_SOCK}" \
  listen=unix:/run/ssh-agent.sock \
  bind=container uid=0 gid=0 mode=0600

incus config set debian environment.SSH_AUTH_SOCK=/run/ssh-agent.sock
```

`incus shell` 是 `incus exec @ARGS@ -- su -l` 的别名。`su -l` 会清除继承的 `SSH_AUTH_SOCK`，所以还需要在容器的登录环境中设置它：

```bash
incus exec debian -- sh -c \
  "printf '%s\\n' 'export SSH_AUTH_SOCK=/run/ssh-agent.sock' > /etc/profile.d/ssh-agent.sh"
incus exec debian -- chmod 0644 /etc/profile.d/ssh-agent.sh
```

重启并验证：

```bash
incus restart debian
incus shell debian

echo "$SSH_AUTH_SOCK"
ssh-add -l
ssh -T git@github.com
```

如果容器能列出密钥但 GitHub 返回 `Permission denied (publickey)`，说明转发已经生效，但宿主机 Agent 尚未加载 GitHub 对应的密钥。退出容器后在宿主机执行：

```bash
ssh-add ~/.ssh/id_ed25519
# 或加载实际关联 GitHub 账号的其他私钥
```

之后容器内可直接使用 SSH 地址：

```bash
git clone git@github.com:herosea/omarchy-init.git
```

这种方式只允许容器调用 Agent 完成签名，不会把私钥文件复制到容器。不过，容器中的 `root` 在 Agent 可用期间可以借助其中的密钥进行认证，因此只应对可信容器启用此配置。宿主机用户级 Agent 未启动时，该转发不可用。

删除转发配置：

```bash
incus config device remove debian ssh-agent
incus config unset debian environment.SSH_AUTH_SOCK
incus exec debian -- rm -f /etc/profile.d/ssh-agent.sh
```

## 状态检查

出现问题时依次检查：

```bash
systemctl status incus --no-pager
incus info
incus profile show default
incus storage list
incus network show incusbr0
incus network list-leases incusbr0
incus exec lab -- ip address
incus exec lab -- journalctl -u NetworkManager -b --no-pager
journalctl -u incus -b --no-pager
```
