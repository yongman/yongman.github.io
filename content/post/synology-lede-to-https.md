---
title: 群晖nas软路由实现全站https
categories: [生命不息]
date: 2018-12-05 23:08:37
---

家用宽带从移动转向电信，体验果然不一样了，从此可以告别移动大内网，告别了frp内网穿透，有了公网ip。原来一直在用的http突然完全暴露在公网下，个人nas虽然一般不会被盯上，但是还是感觉不安全，决定将全面https化。

要https化的主要有LEDE软路由、群晖NAS和NAS上的docker服务。

软路由：外网的入口，然后做端口转发，所以LEDE路由器首先要https。

LEDE自带的酷软中心，已经内置了Let's Encrypt自动签发证书，采用的aliddns的api实现。由于路由器上实现了多拨，所以会有多个公网ip，申请证书的时候申请了通配符子域名证书。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/W2HEfj4.png)

自动签发后生成的证书和key文件最后会生成/etc/uhttpd.key和/etc/uhttpd.crt，由于家用宽带基本上都屏蔽了80和443端口，会自动给配置一条路由转发规则，很是贴心。

群晖：将路由上生成的证书和key文件，通过dsm中的安全性中的`证书`选项中新增证书和key，然后导入证书。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/AN829Wc.png)

然后配置系统默认使用新证书。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/SYJSatv.png)

Docker中运行的home assistant: 复制生成的证书到docker的挂载目录下，修改home assistant的configuration.yaml文件，在http下面添加配置：

![](https://raw.githubusercontent.com/yongman/i/img/picgo/KtOEps9.png)

后续：
由于Let's Encrypt证书有效期90天，所以3个月后需要重新renew证书，有时间写个定时任务，renew证书后自动分发给nas和ha。

