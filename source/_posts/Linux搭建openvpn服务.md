---
title: Linux搭建openvpn服务
date: 2023-10-22 21:49:52
tags:
    - linux
    - network
    - proxy
    - vpn
---

## 1. 准备工作

### 1.1 安装 openvpn 以及依赖

```shell
# manjaro
sudo pacman -S openssl openvpn easy-rsa

# centos
sudo yum install -y epel-release
sudo yum install -y lzo openssl pam openvpn easy-rsa iptables-services
```

### 1.2 禁用不必要的服务

禁用 firewalld 和 SELinux。

```shell
# centos
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
```

## 2. 配置 openvpn

### 2.1 通过 easy-ras 生成证书

#### 2.1.1 配置 easy-ras

openvpn 的证书可以通过 easy-rsa 生成，至于为什么网上文章都是通过它生成的，我也不太清楚，后面可以研究一下。

复制一份模板到 openvpn 的目录下，等待后续使用，这里我的 easy-rsa 版本是 `3.0.8`，读者根据自己的版本调整路径名称。

```shell
cp -a /usr/share/easy-rsa/3.0.8 /etc/openvpn/easy-rsa
cp -a /usr/share/doc/easy-rsa-3.0.8/vars.example /etc/openvpn/easy-rsa/vars
```

编辑 vars 文件，调整其中的一些信息

```conf
set_var EASYRSA_REQ_COUNTRY    "CN"
set_var EASYRSA_REQ_PROVINCE   "Beijin"
set_var EASYRSA_REQ_CITY       "Beijin"
set_var EASYRSA_REQ_ORG        "test"
set_var EASYRSA_REQ_EMAIL      "test@qq.com"
set_var EASYRSA_REQ_OU         "Development Dept."
```

**说明**：

- EASYRSA_REQ_COUNTRY  "所在的国家"
- EASYRSA_REQ_PROVINCE "所在的省份"
- EASYRSA_REQ_CITY     "所在的城市"
- EASYRSA_REQ_ORG      "所属的组织"
- EASYRSA_REQ_EMAIL    "邮件地址"
- EASYRSA_REQ_OU       "组织单位，部门"

#### 2.1.2 通过 easy-ras 生成证书

##### 1. 初始化 pki

进入 easy-rsa 脚本所在目录，初始 pki 目录

```shell
cd /etc/openvpn/easy-rsa
./easyrsa init-pki
```

##### 2. 生成 ca 证书

在这里会提示`Enter New CA Key Passphrase`，该密码在后面用于生成证书用。也可以添加参数 `nopass`，设置无密码模式。

```shell
./easyrsa build-ca
```

##### 3. 生成 server 证书

这里的 `openvpn-server` 是我自己设置的名字，读者也可以指定其他的名字，或者不指定也行。注意我这里使用 **nopass** 参数，这样服务端启动服务时就不用输入密码了。创建时需要输入之前的 `CA` 证书 `PEM` 密码。

```shell
./easyrsa build-server-full openvpn-server nopass
```

##### 4. 生成 dh 文件

生成 `Diffie-Hellman` 算法需要的密钥文件，至于这个文件是干什么用的，我也不知道，后面研究。

```shell
./easyrsa gen-dh
```

##### 5. 生成 client 证书

客户端证书建议使用需要密码的版本。

```shell
./easyrsa build-client-full client nopass # 无密码
./easyrsa build-client-full test # 需要输入密码
```

### 2.2 配置 openvpn-server

#### 2.2.1 复制需要的文件

前面我们已经生成的需要的证书文件，这里我就需要将他们复制对应目录下

```shell
cp /etc/openvpn/easy-rsa/pki/ca.crt /etc/openvpn/server
cp /etc/openvpn/easy-rsa/pki/private/ca.key /etc/openvpn/server
cp /etc/openvpn/easy-rsa/pki/private/openvpn-server.key /etc/openvpn/server
cp /etc/openvpn/easy-rsa/pki/issued/openvpn-server.crt /etc/openvpn/server
cp /etc/openvpn/easy-rsa/pki/dh.pem /etc/openvpn/server
```

#### 2.2.2 生成 ta.key

```shell
cd /etc/openvpn/server
openvpn --genkey --secret ta.key
```

tls-auth key，为了防止 DDOS 和 TLS 攻击，属于可选安全配置。如果配置文件中启用此项（默认是启用的），就需要执行上述命令。配置文件中服务端第二个参数为 0，同时客户端也要有此文件，且 client.conf 中此指令的第二个参数需要为 1。

#### 2.2.3 调整 server.conf

同样，我们需要复制一份模板到 `/etc/openvpn/server` 目录下，这里我的 openvpn 版本是 `2.4.12`，读者根据自己的版本调整路径名称。

```shell
cp /usr/share/doc/openvpn-2.4.12/sample/sample-config-files/server.conf /etc/openvpn/server
```

其配置文件的详细：

```conf
# 表示openvpn服务端的监听地址
local 0.0.0.0

# 监听的端口，默认是1194，最好修改了
port 1194

# 使用的协议，有udp和tcp。建议选择tcp
proto tcp

# 使用三层路由IP隧道(tun)还是二层以太网隧道(tap)。一般都使用tun
dev tun

# ca证书、服务端证书、服务端密钥和密钥交换文件。
# 如果它们和server.conf在同一个目录下则可以不写绝对路径，否则需要写绝对路径调用
ca ca.crt
cert server.crt
key server.key
dh dh.pem

# vpn 服务端为自己和客户端分配IP的地址池。
# 服务端自己获取网段的第一个地址(此处为10.8.0.1)，后为客户端分配其他的可用地址。
# 以后客户端就可以和10.8.0.1进行通信。
# 注意：该网段地址池不要和已有网段冲突或重复，其实一般来说是不用改的。除非当前内网使用了10.8.0.0/24的网段。
server 10.8.0.0 255.255.255.0

# 使用一个文件记录已分配虚拟IP的客户端和虚拟IP的对应关系，
# 以后 openvpn 重启时，将可以按照此文件继续为对应的客户端分配此前相同的IP。也就是自动续借IP的意思。
ifconfig-pool-persist ipp.txt

# 使用tap模式的时候考虑此选项。
# server-bridge XXXXXX

# vpn服务端向客户端推送vpn服务端内网网段的路由配置，以便让客户端能够找到服务端内网。多条路由就写多个Push指令
;push "route 10.0.10.0 255.255.255.0"
;push "route 192.168.10.0 255.255.255.0"

# 让vpn客户端之间可以互相看见对方，即能互相通信。默认情况客户端只能看到服务端一个人；
# 默认是注释的，不能客户端之间相互看见
client-to-client

# 允许多个客户端使用同一个VPN帐号连接服务端
# 默认是注释的，不支持多个客户登录一个账号
duplicate-cn

# 每10秒ping一次，120秒后没收到ping就说明对方挂了
keepalive 10 120

# 加强认证方式，防攻击。如果配置文件中启用此项(默认是启用的)
# 需要执行 openvpn --genkey --secret ta.key，并把 ta.key 放到 /etc/openvpn/server 目录
# 服务端第二个参数为 0；同时客户端也要有此文件，且 client.conf 中此指令的第二个参数需要为 1。
tls-auth ta.key 0

# 选择一个密码。如果在服务器上使用了 cipher 选项，那么您也必须在这里指定它。注意，v2.4客户端/服务器将在TLS模式下自动协商 AES-256-GCM。
cipher AES-256-CBC

# openvpn 2.4版本的vpn才能设置此选项。表示服务端启用lz4的压缩功能，传输数据给客户端时会压缩数据包。
# Push后在客户端也配置启用lz4的压缩功能，向服务端发数据时也会压缩。如果是2.4版本以下的老版本，则使用用comp-lzo指令
compress lz4-v2
push "compress lz4-v2"

# 启用lzo数据压缩格式。此指令用于低于2.4版本的老版本。且如果服务端配置了该指令，客户端也必须要配置
;comp-lzo

# 并发客户端的连接数
max-clients 100

# 通过 ping 得知超时时，当重启vpn后将使用同一个密钥文件以及保持tun连接状态
persist-key
persist-tun

# 在文件中输出当前的连接信息，每分钟截断并重写一次该文件
status openvpn-status.log

# 默认vpn的日志会记录到rsyslog中，使用这两个选项可以改变。
# log指令表示每次启动vpn时覆盖式记录到指定日志文件中，
# log-append则表示每次启动vpn时追加式的记录到指定日志中。
# 但两者只能选其一，或者不选时记录到rsyslog中
;log openvpn.log
;log-append openvpn.log

# 日志记录的详细级别。
verb 3

# 沉默的重复信息。最多20条相同消息类别的连续消息将输出到日志。
;mute 20

# 当服务器重新启动时，通知客户端，以便它可以自动重新连接。仅在UDP协议是可用
;explicit-exit-notify 1
```

实际使用配置：

```shell
egrep -v "^$|^#|^;" server.conf
```

输出

```conf
local 0.0.0.0
port 61194
proto tcp
dev tun
ca ca.crt
cert openvpn-server.crt
key openvpn-server.key
dh dh.pem
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
client-to-client
duplicate-cn
keepalive 10 120
tls-auth ta.key 0
cipher AES-256-CBC
compress lz4-v2
push "compress lz4-v2"
max-clients 100
persist-key
persist-tun
status openvpn-status.log
verb 3
```

#### 2.2.4 启动服务

```shell
# 启动服务
systemctl start openvpn-server@server
# 设置开机自启动
systemctl enable openvpn-server@server
```

### 2.3 配置防火墙

启用 IPv4 转发。

```shell
vi /etc/sysctl.conf
```

添加或者调整如下内容：

```conf
net.ipv4.ip_forward = 1
```

更新配置

```shell
sysctl -p
```

在 iptables 中添加 nat 规则

```shell
# 添加规则
iptables -t nat -A POSTROUTING -s 172.16.0.0/16 -j MASQUERADE iptables -nL -t nat
# 查看规则
iptables -nL -t nat
# 持久化保存
service iptables save
```

### 2.4 配置 openvpn-client

#### 2.4.1 复制文件

同样，我们需要复制一份模板到 `/etc/openvpn/client` 目录下。

```shell
cp /usr/share/doc/openvpn-2.4.12/sample/sample-config-files/client.conf /etc/openvpn/client
```

记得将我们在服务器上生成的 client 证书别忘了下载下来，同样放置到 `/etc/openvpn/client` 目录下。

#### 2.4.2 调整 client.conf

windows 为 client.ovpn，Linux 为 client.conf。
client.conf 配置文件的详细：

```conf
# 标识这是个客户端
client

# 使用三层路由IP隧道(tun)还是二层以太网隧道(tap)。
# 需要和服务端保持一致
dev tun

# 使用的协议，有udp和tcp。
# 需要和服务端保持一致
proto tcp

# 服务端的地址和端口
remote 1.2.3.4 1194

# 一直尝试解析OpenVPN服务器的主机名。
# 在机器上非常有用，不是永久连接到互联网，如笔记本电脑。
resolv-retry infinite

# 大多数客户机不需要绑定到特定的本地端口号。
nobind

# 初始化后的降级特权(仅非windows)
;user nobody
;group nobody

# 尝试在重新启动时保留某些状态。
persist-key
persist-tun

# ca证书、客户端证书、客户端密钥
# 如果它们和client.conf或client.ovpn在同一个目录下则可以不写绝对路径，否则需要写绝对路径调用
ca ca.crt
cert client.crt
key client.key

# 通过检查certicate是否具有正确的密钥使用设置来验证服务器证书。
remote-cert-tls server

# 加强认证方式，防攻击。服务端有配置，则客户端必须有
tls-auth ta.key 1

# 选择一个密码。如果在服务器上使用了cipher选项，那么您也必须在这里指定它。注意，v2.4客户端/服务器将在TLS模式下自动协商AES-256-GCM。
cipher AES-256-CBC

# 服务端用的什么，客户端就用的什么
# 表示客户端启用lz4的压缩功能，传输数据给客户端时会压缩数据包。
compress lz4-v2

# 日志级别
verb 3

# 沉默的重复信息。最多20条相同消息类别的连续消息将输出到日志。
;mute 20
```

有效配置：

```conf
client
dev tun
proto tcp
remote 1.2.3.4 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert test.crt
key test.key
remote-cert-tls server
tls-auth ta.key 1
cipher AES-256-CBC
compress lz4-v2
verb 3
```

#### 2.4.3 启动服务

```shell
systemctl start openvpn-client@client
```

### 2.5 测试连通

```shell
# ping 网关
> ping 10.8.0.1  
PING 10.8.0.1 (10.8.0.1) 56(84) 字节的数据。
64 字节，来自 10.8.0.1: icmp_seq=1 ttl=64 时间=7.84 毫秒
64 字节，来自 10.8.0.1: icmp_seq=2 ttl=64 时间=7.80 毫秒
64 字节，来自 10.8.0.1: icmp_seq=3 ttl=64 时间=8.24 毫秒
```

## 3. 参考

1. [CentOS 7 安装配置 OpenVPN 服务器](https://www.simaek.com/archives/203/)
2. [解决 OpenVPN 客户端所有网络全走 VPN 的问题](https://www.ilxqx.com/archives/jie-jue-openvpn-ke-hu-duan-suo-you-wang-luo-quan-zou-vpn-de-wen-ti)
