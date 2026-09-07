# 终端共享 SSH Agent

本文记录如何在 Omarchy 中使用 systemd 管理的 OpenSSH Agent，让通过 `Super + Enter` 新开的所有终端共享已解锁的 SSH 密钥，避免每次 `git push` 都重复输入密钥口令。

## 前提

确认 OpenSSH 已安装，并且已有 SSH 私钥。本文以 `~/.ssh/id_ed25519` 为例；如果使用其他私钥，需替换为实际路径。

```bash
pacman -Q openssh
ls -l ~/.ssh/*.pub
```

Omarchy 所用的 Arch Linux 已提供 `/usr/lib/systemd/user/ssh-agent.socket` 和对应服务，无需在每个 shell 中手动启动新的 `ssh-agent`。

## 启用用户级 Agent

启用并立即启动 systemd 用户 socket：

```bash
systemctl --user enable --now ssh-agent.socket
```

在 `~/.bashrc` 末尾添加：

```bash
# Share the systemd-managed SSH agent across terminal sessions.
export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/ssh-agent.socket"
```

不要把以下命令写进 `~/.bashrc`：

```bash
eval "$(ssh-agent -s)"
```

它会让每个新终端分别启动 Agent，终端之间无法共享已经解锁的密钥，还会遗留多个 Agent 进程。

## 加载密钥

重新打开一个终端，然后加载私钥：

```bash
ssh-add ~/.ssh/id_ed25519
```

输入一次密钥口令后，同一登录会话中通过 `Super + Enter` 新开的其他终端都会使用同一个 Agent。注销或重启后通常需要重新加载一次密钥，这是预期的安全边界。

## 验证

```bash
systemctl --user is-enabled ssh-agent.socket
systemctl --user is-active ssh-agent.socket
printf '%s\n' "$SSH_AUTH_SOCK"
ssh-add -l
ssh -T git@github.com
```

预期结果：

- socket 状态分别为 `enabled` 和 `active`；
- `SSH_AUTH_SOCK` 为 `$XDG_RUNTIME_DIR/ssh-agent.socket` 展开后的路径；
- `ssh-add -l` 能列出已加载密钥；
- 使用 SSH 远程地址执行 `git push` 时，本次登录期间不再反复要求解锁同一密钥。

普通 `git commit` 不访问 SSH Agent。若口令提示出现在提交操作之后，通常是编辑器启用了提交后自动同步或推送。

## 本机备份

首次配置时已保存原始 Bash 配置：

```text
~/.bashrc.bak-codex-20260903
```
