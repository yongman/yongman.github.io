---
title: DNS劫持和污染原理
categories: [技术记录]
date: 2019-03-27 10:30:23
---

之前一直用的imgur图床最近几天突然打不开了，浏览器打开提示证书不对，证书是facebook的证书，看来是dns把域名解析到facebook的服务器了，因为facebook的地址已经被封锁，这就实现了间接封锁imgur。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/qDckBa8.png)

![](https://raw.githubusercontent.com/yongman/i/img/picgo/KKbOFfj.png)

![](https://raw.githubusercontent.com/yongman/i/img/picgo/18Hs8hf.png)

![](https://raw.githubusercontent.com/yongman/i/img/picgo/xwYrFe8.png)

![](https://raw.githubusercontent.com/yongman/i/img/picgo/pvTO32c.png)

可以看到国内的dns解析都指向了facebook服务器地址，而国外的dns解析都是正常的。

- DNS劫持是DNS服务器中的记录真正的被修改为错误记录。

- DNS污染是DNS服务器中的记录是正确的，但是在链路上有其他的设备会应答回复错误的DNS数据包，导致查询结果被污染。

**DNS劫持实现**

网络运营商处于某些目的或者应政府要求，对DNS服务器进行了某些人为操作，就导致了使用ISP提供的DNS服务器无法将域名正确解析到IP地址。

**DNS污染实现**

因为DNS使用的UDP实现，并且传输的都是未加密的数据包，所以在客户端向DNS服务器发送查询报文，中间的路由器、交换机和网关等设备是可以进行数据包解析，旁路设备可以立刻发送一个回复报文，而报文的内容是可以任意伪造，并非是DNS服务器的正确结果，这就是DNS污染。客户端可能会收到多个回复报文，而先到达的报文被接受，晚到的报文被丢弃，由于延迟，而往往被丢弃的才是正确的结果，从而导致DNS污染的发生。

**解决方法**

对于DNS劫持，只需要将本机的dns服务器指向权威的dns即可，比如谷歌的8.8.8.8。

对于DNS污染就是将DNS报文加密传输，采用代理将dns请求通过加密链路转发到国外的主机，然后在主机上的解析结果再加密传输回客户端。

上面的两种方式也都是对个人来说是很容易解决的，但是如果像这种公共资源被dns劫持，因为广大的用户不可能全部都自己配置dns或者使用代理访问，这就没有很好的解决办法了，对于这种imgur图床被劫持，貌似也只能进行图床迁移了。



**参考**

1. <https://en.wikipedia.org/wiki/DNS_spoofing>
2. [https://baike.baidu.com/item/DNS污染](https://baike.baidu.com/item/DNS%E6%B1%A1%E6%9F%93)
3. <https://pxx.io/2014/08/24/is-a-domain-polluted.html>

