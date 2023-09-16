---
title: Linux 搭建 frp 服务实现内网穿透
date: 2023-09-16 19:51:34
tags:
  - linux
  - network
  - frp
---

## 1. 安装 frp

### 1.1 从源安装

#### 1.1.1 Arch

```shell
pacman -S frpc frps
```

`frpc` 是客户端软件，`frpc` 文件列表：

```shell
> pacman -Ql frpc
frpc /etc/
frpc /etc/frp/
frpc /etc/frp/frpc.ini
frpc /etc/frp/frpc_full.ini
frpc /usr/
frpc /usr/bin/
frpc /usr/bin/frpc
frpc /usr/lib/
frpc /usr/lib/systemd/
frpc /usr/lib/systemd/system/
frpc /usr/lib/systemd/system/frpc.service
frpc /usr/lib/systemd/system/frpc@.service
```

`frps` 是服务端软件，`frps` 文件列表：

```shell
> pacman -Ql frps
frps /etc/
frps /etc/frp/
frps /etc/frp/frps.ini
frps /etc/frp/frps_full.ini
frps /usr/
frps /usr/bin/
frps /usr/bin/frps
frps /usr/lib/
frps /usr/lib/systemd/
frps /usr/lib/systemd/system/
frps /usr/lib/systemd/system/frps.service
frps /usr/lib/systemd/system/frps@.service
```

### 1.2 手动安装

从 [https://github.com/fatedier/frp/releases](https://github.com/fatedier/frp/releases) 上下载最新的 frp 包

```shell
# 下载 文件
wget https://github.com/fatedier/frp/releases/download/v0.51.3/frp_0.51.3_linux_amd64.tar.gz

# 复制文件到 /opt/frp 中
tar -xvf frp_0.51.3_linux_amd64.tar.gz
mkdir /opt/frp
cp frp_0.51.3_linux_amd64/* /opt/frp
```

配置 `frps` `systemd` 服务，创建文件 `/usr/lib/systemd/system/frps.service`

```service
[Unit]  
Description=Frp Server Service  
After=network-online.target  
Wants=network-online.target  
  
[Service]  
Type=simple  
User=nobody  
Restart=on-failure  
RestartSec=5s  
ExecStart=/opt/frp/frps -c /frp/frp/frps.ini  
  
[Install]  
WantedBy=multi-user.target
```

配置 `frpc` `systemd` 服务，创建文件 `/usr/lib/systemd/system/frpc.service`

```service
[Unit]
Description=Frp Client Service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=nobody
Restart=on-failure
RestartSec=5s
ExecStart=/opt/frp/frpc -c /opt/frp/frpc.ini
ExecReload=/opt/frp/frpc reload -c /opt/frp/frpc.ini

[Install]
WantedBy=multi-user.target
```

设置 `frp` 服务开机自启

```shell
# 刷新 service 文件
systemctl daemon-reload
# 设置 frps 开机自启
systemctl enable frps
# 设置 frpc 开机自启
systemctl enable frpc
```

## 2. 配置 frp

### 2.1 配置 frps

`frps` 也就是 `frp` 的服务端，需要安装和部署在有公网 IP 机器上。
简单使用，只需要将映射端口改为自己想要的端口即可

```shell
cat frps.ini
[common]
bind_port = 7000
```

修改完配置文件后重启一下 `frps` 服务即可

```shell
systemctl restart frps
```

### 2.2 配置 frpc

#### 2.2.1 配置 ssh 穿透

`frpc` 部署在需要被穿透的机器上，修改 `frpc` 的配置文件

```ini
[common]
server_addr = x.x.x.x
server_port = 7000

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000
```

上面 `common/server_addr` 就是你的公网机器 IP，`common/server_port` 就是那台机器上 `frps` bind 的端口，在上一节中我们配置为 `bind_port = 7000`。

其中 `[ssh]` 是随便取的，这里为了表明是 `ssh` 穿透，故起名为 `[ssh]`。

`local_ip` 和 `local_port` 是需要暴露到公网的服务地址和端口。

`remote_port` 表示在 `frp` 服务端监听的端口，访问服务端此端口的流量将会被转发到本地服务对应的端口。

重启本机的 `frpc` 服务

```shell
systemctl restart frpc
```

这时就可以在其他机器上通过服务器访问内网机器上 SSH 服务了。

```shell
ssh -p 6000 [user]@x.x.x.x
```

#### 2.2.2 配置安全 ssh 穿透

通过前面的配置我们可以在任何一台机器上访问到咱们内网机器的 SSH 服务了，带来便利的同时也引入了不安全，网络上的黑客也可以通过该端口攻击我们内网机器，一旦被黑客攻陷了，那么咱们内网将完全暴露在黑客面前，这是非常可怕的事情，这里介绍一种相对更安全一点的方式来做 ssh 穿透。

服务端的配置不变，我们需要调整客户端的配置

```ini
[common]
server_addr = x.x.x.x
server_port = 7000

[secret_ssh]
type = stcp
sk = abcdefg
local_ip = 127.0.0.1
local_port = 22
```

其中 `secret_ssh/sk` 设置了访问限制名。

在需要访问内网的机器上也安装 `frpc`，并配置

```ini
[common]
server_addr = x.x.x.x
server_port = 7000

[secret_ssh_visitor]
type = stcp
role = visitor
server_name = secret_ssh
sk = abcdefg
bind_port = 6000
```

其中 `secret_ssh_visitor/sk` 配置为和内网机器上相同的值，该值就是用来限制访问的。

这时在该机器上通过如下命令即可访问内网机器的 SSH 服务了

```shell
ssh -p 6000 [user]@127.0.0.1
```

#### 2.2.3 配置 P2P 模式

`frp` 提供了一种新的代理类型 `xtcp` 用于应对在希望传输大量数据且流量不经过服务器的场景。

> Note that it may not work with all types of NAT devices. You might want to fallback to stcp if xtcp doesn't work.

内网机器上配置：

```ini
# frpc.ini
[common]
server_addr = x.x.x.x
server_port = 7000
# set up a new stun server if the default one is not available.
# nat_hole_stun_server = xxx

[p2p_ssh]
type = xtcp
sk = abcdefg
local_ip = 127.0.0.1
local_port = 22
```

需要访问内网机器上配置

```ini
# frpc.ini
[common]
server_addr = x.x.x.x
server_port = 7000
# set up a new stun server if the default one is not available.
# nat_hole_stun_server = xxx

[p2p_ssh_visitor]
type = xtcp
role = visitor
server_name = p2p_ssh
sk = abcdefg
bind_addr = 127.0.0.1
bind_port = 6000
# when automatic tunnel persistence is required, set it to true
keep_tunnel_open = false
```

这种连接不稳定，有时需要多连几次才能连上。

## 3. 杂乱

查看 `frp` 服务日志

```shell
# 查看 frpc 日志
journalctl -u frps.service -f
# 查看 frpc 日志
journalctl -u frpc.service -f
```

## 4. 参考

1. [https://github.com/fatedier/frp](https://github.com/fatedier/frp)
2. [https://frps.cn/11.html](https://frps.cn/11.html)
