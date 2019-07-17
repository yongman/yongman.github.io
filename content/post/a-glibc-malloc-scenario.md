---
title: 一个glibc内存分配场景分析
categories: [技术记录]
date: 2019-04-25 10:53:38
---



最近看到了2014年关于twemproxy中的issue中有人提到采用pipeline方式大量请求会导致twemproxy占用大量内存的讨论，这个问题在几年前也曾经尝试解决，未果而终。不妨深入了解一下。

因为twemproxy自己对内存池管理来避免频繁申请内存，并且内存池只能增长不能自动缩小，缺乏一定的健壮性和防攻击性。并且看了代码后，知道主要的内存占用就是mbuf内存块，一个mbuf大小默认是16384，于是尝试直接将mbuf入池的操作改为释放mbuf，按理说，在压力过后，mbuf会全部释放，但是测试结果并没有，内存还是被大量占用，操作系统看来内存根本没有释放。

当尝试把msg缓存池也一并释放，结果是内存被完全释放了。后来有尝试mbuf不释放，只释放msg，结果也是内存不释放，只有msg和mbuf一并释放，内存才会被操作系统回收。

因为默认采用glibc中的malloc，malloc会调用brk或mmap系统调用来分配内存，默认大于128k的会通过mmap来申请，其他的会采用brk来申请。内部也通过arena来进行内存的管理，并且有main arena和thread arena。当调用free时，内存也不是直接交还操作系统，而是交给glic进行管理，arena就是管理的原信息。

在twemproxy中mbuf的申请和msg的申请是间隔申请的，并且都不会大于128k，所以内存都是通过brk申请的堆内存，当只尝试释放掉所有的mbuf内存后，在操作系统看来是通过brk申请的内存中出现了非常多的空洞，而这些空洞无法被操作系统回收，因为brk是单向增长或递减，高地址内存不释放，低地址空间无法被回收，这就导致malloc产生内存碎片。



**参考**

1. <https://github.com/twitter/twemproxy/issues/203>
2. [https://wooyun.js.org/drops/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3%20glibc%20malloc.html](https://wooyun.js.org/drops/深入理解 glibc malloc.html)
3. <https://www.cnblogs.com/zhaoyl/p/3820852.html>
