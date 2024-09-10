---
title: OpenVPN server 反向访问 client 端局域网
categories: 
- [生命不息]
date: 2024-09-09 23:53:38
---

## 1. 背景需求

需求是内网穿透，想把本地局域网和国外的 VPS 主机组成局域网，连接 OpenVPN 后，VPS 主机可以直接访问本地局域网，如 `192.168.31.1/24`，并且由于 VPS 拥有稳定的 IP，所以请求的链路是 OpenVPN server 端访问 client 端的局域网。使用 `zerotier` 也能实现这种需求，但是最近不太稳定，大概是因为运营商对 UDP 执行 QoS 导致丢包严重，所以要使用 TCP 协议来建立隧道。

## 2. 安装 OpenVPN

分别在服务端和本地机器一键安装 OpenVPN，脚本开源，无黑科技，可放心使用。

```
wget https://git.io/vpn -O openvpn-install.sh && sudo bash openvpn-install.sh
```

按照提示一步一步安装，需要注意的是，安装的时候选择 TCP 协议。

## 3. 配置 Server 端

在上面的脚本安装完成后，配置和证书等繁琐配置已经自动安装完成，只需调整一下配置。

### 在 Linux 系统中，可以使用以下命令启用 IP 转发：

```
sysctl -w net.ipv4.ip_forward=1
```

永久启用 IP 转发，需要修改 /etc/sysctl.conf 文件，取消以下行的注释：

```
net.ipv4.ip_forward=1
```

### 更新 server.conf 配置

```
local 172.24.10.88 #local 是 server 的本地局域网地址
port 12294
proto tcp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
auth SHA512
tls-crypt tc.key
topology subnet
push "route 172.24.10.88 255.255.255.255" # 向客户端推送本地局域网的路由，客户端访问 172.24.10.88 会走 vpn 网络
route 192.168.31.0 255.255.255.0
server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt
keepalive 10 120
user nobody
group nogroup
persist-key
persist-tun
verb 3
crl-verify crl.pem
client-config-dir /etc/openvpn/server/ccd # 客户端配置文件，注意具体的配置文件名需要和安装时填写的 client 一致。
```

### CCD 文件配置

```
ifconfig-push 10.8.0.2 255.255.255.0 # 向客户端推送路由配置，配置客户端的 IP 地址
iroute 192.168.31.0 255.255.255.0 # 这条指令用于在 OpenVPN 服务器上声明客户端可以路由到的内部网络。通过这条指令，OpenVPN 服务器知道应该将发往 192.168.31.0/24 网络的流量通过该客户端进行路由。
```

### 重新启动服务端

```
sudo service openvpn-server@server restart
```

## 4. 配置客户端

### 开启 IP 内核转发

用于通过该机器访问局域网的其他机器

### 配置调整

直接复制在 2 安装完成后生成的客户端配置，然后稍微进行调整

```
client
dev tun
proto tcp
remote vps-ip port
resolv-retry infinite
nobind
persist-key
persist-tun
remote-cert-tls server
auth SHA512
ignore-unknown-option block-outside-dns
push "route 192.168.31.0 255.255.255.0" # 给服务推送路由，把本地局域网网段推送给 server 端，server 访问改网段会通过 vpn 转发给本客户端
verb 3
<ca>
-----BEGIN CERTIFICATE-----
x
-----END CERTIFICATE-----
</ca>
<cert>
-----BEGIN CERTIFICATE-----
x
-----END CERTIFICATE-----
</cert>
<key>
-----BEGIN PRIVATE KEY-----
x
-----END PRIVATE KEY-----
</key>
<tls-crypt>
-----BEGIN OpenVPN Static key V1-----
x
-----END OpenVPN Static key V1-----
</tls-crypt>
```

### 启动客户端

```
sudo openvpn --config /etc/openvpn/client/aws.ovpn --log-append /var/log/openvpn.log
```

## 验证

在服务端执行 `ping 192.168.31.1` 能够访问，配置成功。
