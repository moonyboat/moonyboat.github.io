---
title: "Tailscale SSH配置笔记"
date: 2026-03-16T16:36:00+08:00
draft: false
tags: ["Tailscale", "SSH"]
summary: "  "
---

![](images/Pasted%20image%2020260316163613.png)

# 整体架构

```txt
本地电脑 (Win Mac...)  
    │  
    │  
Tailscale 网络  
    │  
    │  
Ubuntu 服务器  
    │  
    ├─ SSH  
    ├─ VSCode Remote  
    └─ AI训练 / 深度学习
```

## 1. 安装 SSH 服务

在 Ubuntu 上安装 OpenSSH Server：

```bash
sudo apt update  
sudo apt install openssh-server
```

## 2. 启动 SSH 服务

启动 SSH：`sudo systemctl start ssh`
设置开机自动启动：`sudo systemctl enable ssh`
打开 SSH：`sudo systemctl enable --now ssh`

## 3. 检查 SSH 服务状态

```bash
sudo systemctl status ssh
```

正常状态为Active: active (running)，或者可以查看ssh自己看能不能连通

```bash
ssh localhost
ssh username@127.0.0.1
```

顺便检查一下防火墙UFW状态，如果启用，放行 SSH，22为端口号。

```bash
sudo ufw status
sudo ufw allow 22/tcp
sudo ufw allow 2222/tcp
```

## 4. 查看服务器 IP 地址

```bash
hostname -I
# 示例  10.168.1.101 ...
# 如果两个主机连接在同一个局域网，可以直接连接
ssh username@10.168.1.101
```

# 非局域网远程连接方案

如果不在同一个局域网，可以使用 **Tailscale 内网穿透**。

## 5. 安装 Tailscale
```bash
# Ubuntu上安装
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
# Windows上直接下载软件即可
# 网站：https://tailscale.com/download

# 然后登入到同一个账号！

# 查看Tailscale IP，也可以在浏览器直接查看
tailscale ip -4
# 例如100.xxx.xxx.xxx，这个 IP 用于远程 SSH
```
## 6. SSH 端口问题
```bash
# 检查 SSH 监听端口
sudo ss -ntlp | grep ssh
```
示例输出：`LISTEN 0 128 0.0.0.0:2222`，说明 SSH **运行在 2222 端口**；如果是其他数字，可能就是不一样的端口号。

配置文件在`/etc/ssh/sshd_config`，关键配置为
```bash
Port 2222  
PermitRootLogin no
```
## 7. 远程连接 SSH
```bash
ssh -p 2222 username@100.xxx.xxx.xxx
```
首次连接可能会出现The authenticity of host can't be established. Are you sure you want to continue connecting?直接输入yes即可。SSH 会把服务器指纹记录到`~/.ssh/known_hosts`

## 8. VSCode 远程SSH
安装扩展：Remote - SSH
Windows 的SSH配置文件：`C:\Users\用户名\.ssh\config`
```txt
Host username（自取）  
    HostName 100.xxx.xxx.xxx  
    User username(服务器的用户名)  
    Port 2222
```
## 9. SSH 免密码登录
SSH连接：`ssh -p 2222 x@100.xxx.xxx.xxx`
生成密钥：`ssh-keygen`
复制公钥到服务器：`ssh-copy-id -p 2222 username@100.xxx.xxx.xxx`
以后连接不需要输入密码。
