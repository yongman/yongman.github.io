---
title: HyperLogLog - 基数统计算法
categories: [算法]
date: 2018-08-09 19:29:22
---

![](https://raw.githubusercontent.com/yongman/i/img/picgo/mjiCDPY.jpg)

如果有一个需求，就是要统计一个全球性活跃网站的访问UV，根据用户ip识别。

当然可以是要set数据结构，把所有的ip都塞到set数据结构中，最后取一下set的大小。
但是如果只是想要做一些大数据统计，而不关心具体单个的用户信息，如果有上亿用户，这个统计可能需要高达`2G`的内存，有必要保存每个用户的ip地址？没有。

## 基数估计

HyperLogLog是一种基数统计算法，算法的神奇之处是充分发挥数学之美，在保证相对误差的前提下，用极少的内存完成了同样的工作。redis实现中，`12KB`内存就可以完成`2^64`个数据的统计。

HyperLogLog实现的原理基本思想是利用数字的bit信息中第一个`1`出现的位置预估整体基数，并采用分桶的方式来减少误差。

`antirez`举了个形象的例子：

基数估计是如何工作的

> Here I’ll cover only the basic idea using a very clever example found at [3]. Imagine you tell me you spent your day flipping a coin, counting how many times you encountered a non interrupted run of heads. If you tell me that the maximum run was of 3 heads, I can imagine that you did not really flipped the coin a lot of times. If instead your longest run was 13, you probably spent a lot of time flipping the coin.

如何减少估算的误差

> However if you get lucky and the first time you get 10 heads, an event that is unlikely but possible, and then stop flipping your coin, I’ll provide you a very wrong approximation of the time you spent flipping the coin. So I may ask you to repeat the experiment, but this time using 10 coins, and 10 different piece of papers, one per coin, where you record the longest run of heads. This time since I can observe more data, my estimation will be better.

## 实现原理

数据的存储的结构如上图（*该实例并非redis中的实现*）
![](https://raw.githubusercontent.com/yongman/i/img/picgo/Zi5nEEV.png)

1. 对于输入，计算hash
2. 去hash后的二进制比特最右6bit来进行分桶，桶的个数是2^6=64个，对应的桶是6
3. 计算的hash值左侧的18位用来进行比特统计，从右侧数第一个1bit位的位置为3
4. 设置第6个register的值为6（由于原来是0，6大于0，所以更新。如果原来的值比6大，则不更新）
5. 完成元素添加

读取的过程需要进行基数估算和修正，当然返回的结果是有误差率的。

不行，看到数据公式有点头大，先休息会吧。 :-)。

图示地址：[HyperLogLog](http://content.research.neustar.biz/blog/hll.html)

### 参考
1. [Redis new data structure: the HyperLogLog](http://antirez.com/news/75)
2. [HyperLogLog算法详解 - YoonPer](http://www.yoonper.com/post.php?id=79)
3. [HyperLogLog — Cornerstone of a Big Data Infrastructure](https://research.neustar.biz/2012/10/25/sketch-of-the-day-hyperloglog-cornerstone-of-a-big-data-infrastructure/)
