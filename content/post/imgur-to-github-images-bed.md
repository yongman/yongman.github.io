---
title: imgur图床迁移github图床
categories: [技术记录]
date: 2019-03-28 15:58:20
---

因为[imgur被dns劫持](https://xiking.win/2019/03/27/dns-cache-pollution/)，所以blog图片都裂了，没有好的办法，只能找其他的替代图床。之前也一直不忍心使用github在用作图床，毕竟github是用作代码托管和IT社区，不以代码分享的使用就是滥用。

但是在[V2EX](https://www.v2ex.com/t/299197)上看到几年前有个使用github做图床的repo作者询问了github官方，给出的答复这种使用方式不属于滥用，是允许的。

```
> Thanks for your question! We've reviewed your project and, in addition to uploading files, it appears to assist in generating rawgit URLs. Is that correct? 
> 
> If that's the case, your project doesn't appear to violate GitHub's Terms of Service, though you may want to check in with the owner of rawgit if you haven't already done so. 
> 
> Of course, any individual who decided to use your code would be responsible for making sure their usage and content didn't violate our Terms. 
> 
> Please let me know if you have any other questions.
```

既然官方给的回复是这种方式是合理的，那就将图床迁移到github了。

### 0. 创建github repo用做图床

- 创建repo，命名`i`，尽量简短url长度
- 设置token
- 配置PicGo，分支使用非master分支，避免让github统计成contribution，点亮小绿星。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/20190328174823.png)

###  1. 下载原图

- 修改hosts

由于i.imgur.com被dns污染，通过未污染的dns查询正确的地址，加入hosts文件。

```
dig @8.8.8.8 i.imgur.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> @8.8.8.8 i.imgur.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16557
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;i.imgur.com.			IN	A

;; ANSWER SECTION:
i.imgur.com.		8577	IN	CNAME	prod.imgur.map.fastlylb.net.
prod.imgur.map.fastlylb.net. 27	IN	A	151.101.24.193

;; Query time: 43 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Thu Mar 28 17:51:58 CST 2019
;; MSG SIZE  rcvd: 97
```

在hosts文件中添加记录

`151.101.24.193 i.imgur.com `

- 下载图片

`grep "i.imgur.com" ../posts/* | cut -d '(' -f2 | cut -d ')' -f1 | xargs wget`

批量将imgur图片下载，并保留原文件名，这样后面更新图片链接方便了。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/20190328172912.png)

- 上传github

将图片commit后push到刚刚创建的repo。

### 2. markdown链接更新

因为图片的文件名保持不变，只需要更新url的路径前缀即可，使用如下命令完成。

`sed -i 's/i.imgur.com/raw.githubusercontent.com\/yongman\/i\/img\/picgo\//g' * `


### 3. 生成静态页面

`hexo g`大功告成。



**参考**

1. <https://www.jianshu.com/p/87ea6603a824>
2. <https://www.v2ex.com/t/299197>

