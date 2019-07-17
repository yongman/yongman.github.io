---
title: 了解负载均衡
categories: [技术记录]
date: 2018-11-06 19:43:23
---

### 种类

- 硬件负载均衡
  * F5
![](https://raw.githubusercontent.com/yongman/i/img/picgo/PwcW9y7.png)
  * TP-Link 
  
  硬件一般都是定制化，价格比较贵，不太符合互联网公司的价值观，要使用普通pc来完成专业硬件的工作。
![](https://raw.githubusercontent.com/yongman/i/img/picgo/BZSD9rv.png)
Nginx Plus和F5 BIG-IP 11050价格对比。

- 软件负载均衡
  * 四层交换
  * 七层交换

### 软件负载均衡

![](https://raw.githubusercontent.com/yongman/i/img/picgo/YP8n8uw.png)

软件负载均衡按工作的层次可以分为四层和七层，四层就是工作在OSI的四层，工作在TCP，通过三层IP和四层端口号来进行负载均衡；七层交换处理支持四层负载均衡外，只要分析应用层的信息，比如HTTP协议的URI或者cookie等信息来进行负载。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/0eBnupo.png)

#### 常见的四层均衡

四层均衡通过报文中的目标地址和端口，根据配置的负载均衡策略，选择目标服务器。

以tcp为例，负载均衡在收到第一个赖在客户端的SYN请求时，通过修改报文中的地址修改，直接转发给该服务器。tcp连接的建立，是客户端和服务端直接建立的。

- LVS

![](https://raw.githubusercontent.com/yongman/i/img/picgo/W6aFSwK.png)

LVS架构图能看到LVS由三层组成：

1. 流量入口
2. 后端server集群
3. 共享存储

LVS的几种工作模式：

![](https://raw.githubusercontent.com/yongman/i/img/picgo/YLB3I0f.png)
**LVS-NAT**：工作机制类似D-NAT，这种模式下所有的流量都需要经过Balancer，容易成为瓶颈。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/f97NV67.png)
**LVS-DR**：Direct Routing，直接路由，要求Balancer和后端集群在同一个网络环境，需要修改报文中目标的mac地址，然后重新封装，报文中的源ip和目标ip都没有被修改，后端server直接相应客户端，将数据返回，流量不需要经过Balancer，降低Balancer负载。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/o8KWtPm.png)
**LVS-TUN**：类似DR模式，但是后端server和Balancer不在同一个网络，使用IP隧道封装，通过公网发送给后端server，后端server解析报文后直接相应客户端。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/wikQxan.png)
LVS的高可用是通过主备来实现的，当主挂掉后会进行主备切换，达到高可用效果。

- Nginx

Nginx即可作为四层转发，也可以作为七层转发。四层转发tcp配置示例：
```
stream {
  upstream tcp_backend {
    server srv1.example.com:3306;
    server srv2.example.com:3306;
  }
  server {
    listen 3306;
    proxy_pass tcp_backend;
  }
}
```

- Seesaw

基于LVS的负载均衡平台。开源版本非google内部使用版本。

#### 常见七层均衡

- HAProxy

```
listen MaxCDN-HAProxy 10.10.10.10:80
mode http
stats enable
stats uri /haproxy?status
balance roundrobin
server Server01 10.10.10.1:80 check
server Server02 10.10.10.2:80 check
server Backup   10.10.10.3:80 backup
```
有多种负载均衡策略，rr、uri、uri parameter等。

- Nginx

```
upstream backend1 {
  # list of servers
}

upstream backend2 {
  # list of servers
}

location /shop {
  proxy_pass http://backend1;
}

location /blog {
  proxy_pass http://backend2;
}
```
nginx典型通过uri来进行负载均衡。

- 各种应用层proxy，eg. MySQL Proxy

各种数据库分布式proxy服务，mysql proxy、redis proxy等应用层proxy，通过特定请求数据的进行hash计算，定位后端server，请求对应后端server获取数据后返回给客户端。

上面的这些负载均衡方式都是有流量单点，如果单点负载达到极限，就需要进行水平扩展，可以使用DNS或者类似的命名服务水平扩展Balancer。


**参考链接**
1. [5 Reasons to Switch from F5 BIG-IP to NGINX Plus \| NGINX](https://www.nginx.com/blog/5-reasons-switch-f5-big-ip-to-nginx-plus/)
2. [NGINX Plus vs. F5 BIG‑IP: A Price‑Performance Comparison](https://www.nginx.com/blog/nginx-plus-vs-f5-big-ip-a-price-performance-comparison/)
3. [What is Load Balancing? \| DigitalOcean](https://www.digitalocean.com/community/tutorials/what-is-load-balancing)
4. [10 Open Source Load Balancer for HA and Improved Performance](https://geekflare.com/open-source-load-balancer/)
5. [General Architecture of LVS Server Clusters](http://www.linuxvirtualserver.org/architecture.html)
6. [负载均衡集群 LVS 详解（Loadbalancer & LVS）](https://blog.csdn.net/qq_37595946/article/details/77919018)
7. [What Is Layer 4 Load Balancing? \| NGINX Load Balancer](https://www.nginx.com/resources/glossary/layer-4-load-balancing/)

