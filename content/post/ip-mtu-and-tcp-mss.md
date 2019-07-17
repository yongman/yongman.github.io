---
title: IP MTU和TCP MSS
categories: [生命不息]
date: 2019-01-21 11:04:19
---

使用群晖的vmm虚拟lede软路由几个月，在nas下载流量较大的时候会出现lede掉线，多拨不稳定掉线，但是又想使用lede的插件中心，方便配置ss。

所以采用了双软路由，ikuai来进行pppoe多拨，lede用作旁路路由，群晖网络配置ikuai为默认网关，因为nas下载流量较大，直接挂在一级路由下，减少数据包的多一次转发。

使用ikuai提供dhcp服务，dhcp服务中配置默认网关是lede地址，这样其他的设备都会用lede做默认网关，可以利用到lede的多种插件。

简单的拓扑图

![](https://raw.githubusercontent.com/yongman/i/img/picgo/n0EKgbH.png)

在配置完ikuai后，在系统设置中修改了tcp最大包长度，默认是1400，手动修改成了1500，因为其他设备默认的IP MTU设置的是1500，就出现了很怪异的现象，就是ikuai后台本身能够进去，但是经常不稳定，nas、lede的https和ssh都会出现卡死的情况。ssh登录nas，tcpdump抓包。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/dp8TcPI.png)

![](https://raw.githubusercontent.com/yongman/i/img/picgo/dlzT2OV.png)

IP MTU配置了1500，数据包的格式如下：

![](https://raw.githubusercontent.com/yongman/i/img/picgo/fIMPvfy.png)

在tcpdump抓包可以看到很多这种回复的数据包，并且ack是同一个报文，一直在进行重传，并且长度1436，在ikuai中将MSS设置成了1500，这样在ikuai向外转发时，数据包的IP MTU长度为：1500+20+20=1540，因为网络设置PPPOE默认MTU是1480，所以数据包大小超过MTU，导致了丢包。

解决方式就是将ikuai中的TCP MSS设置成默认值1400，或者取消该设置。



**参考**

1. https://blog.apnic.net/2014/12/15/ip-mtu-and-tcp-mss-missmatch-an-evil-for-network-performance/
2. http://blog.perterpon.com/2017/09/12/why-MTU-equals-1500/