---
title: Linux 安装和配置 clash
date: 2023-09-16 19:47:24
tags: linux, proxy
---

## 1. 安装 clash

### 1.1 从源安装

#### 1.1.1 Arch

```shell
sudo pacman -S clash-git
```

安装后文件列表：

```shell
clash-git /etc/
clash-git /etc/clash/
clash-git /etc/clash/config.yaml
clash-git /usr/
clash-git /usr/bin/
clash-git /usr/bin/clash
clash-git /usr/lib/
clash-git /usr/lib/systemd/
clash-git /usr/lib/systemd/system/
clash-git /usr/lib/systemd/system/clash.service
clash-git /usr/lib/systemd/system/clash@.service
clash-git /usr/lib/sysusers.d/
clash-git /usr/lib/sysusers.d/clash.conf
clash-git /usr/share/
clash-git /usr/share/licenses/
clash-git /usr/share/licenses/clash/
clash-git /usr/share/licenses/clash/LICENSE
```

## 1.2 手动安装

从 github 下载 clash

```shell
wget https://github.com/Dreamacro/clash/releases/download/v1.18.0/clash-linux-amd64-v3-v1.18.0.gz
```

解压 `gzip` 包

```shell
gunzip clash-linux-amd64-v3-v1.18.0.gz
```

复制 clash 到 `/opt` 中，并建立 `/bin` 目录下的软链，方便直接使用。

```shell
mkdir -p /opt/clash /etc/clash
cp clash-linux-amd64-v3-v1.18.0 /opt/clash/clash
ln -s /opt/clash/clash /usr/bin/clash
```

创建 service 文件，方便通过 `systemd` 管理，文件 `/usr/lib/systemd/system/clash.service`

```service
[Unit]
Description=Clash Service
After=network.target nss-lookup.target

[Service]
User=root
ReadWritePaths=/etc/clash
ExecStart=/usr/bin/clash -d /etc/clash
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-abort

[Install]
WantedBy=multi-user.target
```

在低版本（等待确实具体是从哪个版本改动的）的 `systemd` 中，需要将 `ReadWritePaths=/etc/clash` 改为

```shell
ReadWriteDirectories=/etc/clash
```

创建完文件后更新一下 `systemd` 服务

```shell
systemctl daemon-reload
```

在 Arch 中 clash 是以 clash 用户启动，可能是为了安全，后面可以考虑更新一下。

## 2. 配置 clash

创建配置文件

```shell
wget -O /etc/clash/config.yaml {订阅地址}
wget -O /etc/clash/Country.mmdb https://github.com/Dreamacro/maxmind-geoip/releases/download/20230912/Country.mmdb
```

记得将 `config.yaml` 中的 `external-controller: :9090` 管理地址改为 `external-controller: 127.0.0.1:9090`，避免暴露给外网

## 3. 设置 clash

### 3.1 获取当前代理配置

```shell
curl -X GET -H "Content-Type: application/json" http://127.0.0.1:9090/proxies
```

可以使用 `jq` 显示格式后的 json 数据

```shell
curl -X GET -H "Content-Type: application/json" http://127.0.0.1:9090/proxies | jq
```

### 3.2 获取当前节点选择

```shell
$ curl -X GET -H "Content-Type: application/json" http://127.0.0.1:9090/proxies/节点选择
{"alive":true,"all":["手动选择","自动选择","故障切换","DIRECT"],"history":[],"name":"节点选择","now":"自动选择","type":"Selector","udp":true}
```

### 3.2 设置节点选择为自动选择

```shell
curl -X PUT -H "Content-Type: application/json" -d '{"name":"自动选择"}' http://127.0.0.1:9090/proxies/节点选择
```

### 3.3. 获取手动选择的节点

```shell
curl -X GET -H "Content-Type: application/json" http://127.0.0.1:9090/proxies/手动选择 | jq
```

### 3.3 设置手动选择的节点

其中 `xxxxx` 就是选择的节点

```shell
curl -X PUT -H "Content-Type: application/json" -d '{"name":"xxxxx"}' http://127.0.0.1:9090/proxies/手动选择
```

## 4. 杂乱东西

查看 clash 服务的日志

```shell
journalctl -u clash.service -f
```

## 5. 参考

1. [https://github.com/Dreamacro/clash](https://github.com/Dreamacro/clash)
2. [Country.mmdb 提供者](https://github.com/Dreamacro/maxmind-geoip/releases)
3. [The External Controller](https://dreamacro.github.io/clash/runtime/external-controller.html)
4. [使用 Clash-API 切换节点](https://sakronos.github.io/Note/2021/03/06/%E4%BD%BF%E7%94%A8Clash-APIj%E5%88%87%E6%8D%A2%E8%8A%82%E7%82%B9/)
