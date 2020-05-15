---
title: Linux网卡多队列实现
categories: [技术记录]
tags: ["linux","kernel"]
date: 2020-05-15 01:00:00
---

- rps

![](https://raw.githubusercontent.com/yongman/i/img/picgo/20200515173225.png)

![](https://raw.githubusercontent.com/yongman/i/img/picgo/20200515173240.png)

![](https://raw.githubusercontent.com/yongman/i/img/picgo/20200515173132.png)

Receive Package Steering，是用来在软件层面实现网卡报文在多个cpu处理的负载均衡。

对于多队列网卡，硬件能够支持网卡中断的处理cpu核，但是往往cpu的核数比网卡硬件接收队列数量大很多，只依赖硬件队列无法充分发挥cpu多核的性能，rps可以通过软件来实现报文的处理分散。

对于单队列网卡，硬件本身无法将网卡的中断绑定到多cpu核，单核cpu存在性能处理瓶颈，rps可以将报文平均分配多多个cpu上。

以tcp报文为例，根据报文的四元组信息计算出hash值找到对应的cpu，将报文交给对应的cpu backlog进行下一步处理。在backlog队列满后，将backlog加入cpu私有数据待轮询设备列表中并触发cpu软中断，cpu这时会将backlog队列中的报文进行协议栈处理。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/20200515173338.png)

- rfs

Receive Flow Steering，rfs是基于rps的整体逻辑实现的，不同的点是改进了对不同的报文选择哪个cpu，在rps中，cpu的选择只是基于报文的hash值来决定的，虽然性能负载均衡效果好，但是会导致报文处理cpu和应用程序调度在不同的cpu，导致cpu cache miss，影响应用性能。

rfs的目标是在选择报文交给哪个cpu处理时，选择的是应用程序所在的cpu进行内核态报文处理，增加cpu缓存命中率。具体的实现依赖内核中的两个流表：

1）设备流表：上次内核态处理该流中报文的cpu

2）全局socket流表：流中报文期望被处理的目标cpu

- rss

![](https://raw.githubusercontent.com/yongman/i/img/picgo/20200515154351.png)

Receive Side Scaling，网卡多队列硬件实现。


**参考**

1. https://tqr.ink/2017/07/09/implementation-of-rps-and-rfs/
2. https://tonydeng.github.io/sdn-handbook/dpdk/queue.html




