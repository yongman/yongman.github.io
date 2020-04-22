---
title: 记一次Redis超时抖动问题排查
categories: [存储]
tags: ["redis","k8s","wireshark"]
date: 2020-04-21 01:00:00
---

#### 1. 问题现象

业务客户端请求Redis集群客户端看经常出现30-70ms的超时抖动，而执行的命令都是O(1)的复杂度，预期正常返回在0.1ms内，明显不符合预期。

#### 2. 问题分析

- 客户端逻辑？

由于请求链路比较长，客户端接入通过Load Balancer方式接入，然后流量会转发到Envoy proxy，最后流量会转到Redis server。首先怀疑是客户端逻辑问题，于是写了个精简的客户端模拟流量，统计耗时，用于排除客户端业务逻辑导致的长耗时。

```go
package main

import (
	"flag"
	"fmt"
	"github.com/garyburd/redigo/redis"
	"math/rand"
	"time"
)

const charset = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"

var seedRand *rand.Rand = rand.New(rand.NewSource(time.Now().UnixNano()))

var (
	verbose      bool
	keep         bool
	keySuffixLen int
	addr         string
	auth         string
	interval     int
)

func init() {
	flag.StringVar(&addr, "addr", "127.0.0.1:6379", "redis protocol address")
	flag.StringVar(&auth, "auth", "", "auth password")
	flag.BoolVar(&verbose, "verbose", false, "print verbose")
	flag.BoolVar(&keep, "keep", false, "keep tcp connection")
	flag.IntVar(&keySuffixLen, "suffixlen", 10, "random key suffix length, 0 will result to fixed key")
	flag.IntVar(&interval, "interval", 100, "sleep interval to next loop")

}

func main() {
	flag.Parse()
	var c redis.Conn
	var err error

	if keep {
		c, err = redis.DialTimeout("tcp", addr, 10*time.Second, 10*time.Second, 10*time.Second)
		if err != nil {
			fmt.Println("Connect to redis error", err)
		}
		defer c.Close()
	}
	for {
		keySuffix := randStr(keySuffixLen)
		key := "__NOT_EXISTS_KEY__" + keySuffix
		t1 := time.Now()
		if !keep {
			c, err = redis.DialTimeout("tcp", addr, 10*time.Second, 10*time.Second, 10*time.Second)
			if err != nil {
				fmt.Println("Connect to redis error", err)
			}
		}
		t2 := time.Now()
		// send auth
		reply, err := c.Do("AUTH", auth)
		if verbose {
			fmt.Println("AUTH", reply, err)
		}
		t3 := time.Now()
		reply, err = c.Do("GET", key)
		if verbose {
			fmt.Println("GET", key, reply, err)
		}
		t4 := time.Now()
		reply, err = c.Do("DEL", key)
		if verbose {
			fmt.Println("DEL", key, reply, err)
		}
		t5 := time.Now()

		fmt.Println(t5, "Time: connect:", t2.Sub(t1), "auth:", t3.Sub(t2), "key:", key, "get:", t4.Sub(t3), "del:", t5.Sub(t4))
		time.Sleep(time.Duration(interval) * time.Millisecond)

		if !keep {
			c.Close()
		}
	}
}

func randStr(n int) string {
	b := make([]byte, n)
	for i := range b {
		b[i] = charset[seedRand.Intn(len(charset))]
	}
	return string(b)
}
```

客户端逻辑很简单，每个周期依次执行3条命令，`AUTH`，`GET`，`DEL`命令，可以配置使用短连接和长连接、随机key后缀长度等。

但是模拟的结果还是的确存在抖动。

```
2020-04-20 10:43:17.323785872 +0800 CST m=+244368.682148883 Time: connect: 264ns auth: 265.284µs key: __NOT_EXISTS_KEY__ get: 66.206239ms del: 636.014µs
2020-04-20 10:43:19.962171655 +0800 CST m=+244371.320534600 Time: connect: 352ns auth: 9.670946ms key: __NOT_EXISTS_KEY__ get: 2.760777ms del: 313.99µs
2020-04-20 10:43:22.084654316 +0800 CST m=+244373.443017276 Time: connect: 546ns auth: 1.970563ms key: __NOT_EXISTS_KEY__ get: 311.542µs del: 279.83µs
2020-04-20 10:43:24.823786502 +0800 CST m=+244376.182149474 Time: connect: 543ns auth: 213.699µs key: __NOT_EXISTS_KEY__ get: 11.24124ms del: 292.548µs
2020-04-20 10:43:25.15902586 +0800 CST m=+244376.517388822 Time: connect: 211ns auth: 31.658569ms key: __NOT_EXISTS_KEY__ get: 1.21072ms del: 314.163µs
2020-04-20 10:43:26.473398918 +0800 CST m=+244377.831761873 Time: connect: 556ns auth: 1.231028ms key: __NOT_EXISTS_KEY__ get: 456.923µs del: 358.346µs
2020-04-20 10:43:29.825486003 +0800 CST m=+244381.183848972 Time: connect: 561ns auth: 258.122µs key: __NOT_EXISTS_KEY__ get: 18.612177ms del: 313.134µs
2020-04-20 10:43:32.249667012 +0800 CST m=+244383.608029961 Time: connect: 476ns auth: 259.625µs key: __NOT_EXISTS_KEY__ get: 1.134468ms del: 308.41µs
2020-04-20 10:43:33.423422354 +0800 CST m=+244384.781785325 Time: connect: 207ns auth: 220.482µs key: __NOT_EXISTS_KEY__ get: 63.491164ms del: 391.856µs
2020-04-20 10:43:34.923937713 +0800 CST m=+244386.282300668 Time: connect: 537ns auth: 281.486µs key: __NOT_EXISTS_KEY__ get: 85.805177ms del: 654.436µs
2020-04-20 10:43:35.758585584 +0800 CST m=+244387.116948538 Time: connect: 208ns auth: 26.512326ms key: __NOT_EXISTS_KEY__ get: 882.832µs del: 308.356µs
2020-04-20 10:43:38.057318509 +0800 CST m=+244389.415681529 Time: connect: 544ns auth: 371.925µs key: __NOT_EXISTS_KEY__ get: 302.44µs del: 75.936648ms
```

上面是使用长连接的输出结果，只grep出了超过1ms的耗时请求，可以看到，最长抖动长达75ms。

- 机器负载？

不管是宿主机整体负载还是pod中的进程负载，基本上都没有什么压力，排除是高负载导致问题。

- 网络抖动？

由于k8s集群规模比较大，线上也一直在变更，宿主机上经常会有kube-proxy管理的`iptables-restore`命令修改内核iptables，经常持续几秒钟，于是怀疑是`iptables-restore`命令会导致tcp连接时间，搜索一番，没有看到iptables相关操作会导致长连接的请求抖动，并且线下没有复现环境，于是决定先增加ping监控，如果是网络抖动或者iptables的影响，ping包应该也会存在抖动。

但是ping结果对应时间点没有出现抖动现象，排除iptables影响。

- 网络链路?

为了排除中间链路影响，客户端直接连接后端redis，减少链路影响，请求的结果依旧存在长耗时抖动，并且不同的后端节点都存在不同程度的抖动现象，似乎和宿主机node也没有太大关系，似乎是一个普遍性存在的问题。

```
2020-04-20 11:36:22.41189801 +0800 CST m=+2.259429352 Time: connect: 448ns auth: 43.278416ms key: __NOT_EXISTS_KEY__chKsAgMJSy get: 141.053µs del: 119.188µs
2020-04-20 11:36:30.812063893 +0800 CST m=+10.659595240 Time: connect: 434ns auth: 46.337946ms key: __NOT_EXISTS_KEY__TuOw6FEstF get: 169.114µs del: 138.87µs
2020-04-20 11:36:42.71191839 +0800 CST m=+22.559449735 Time: connect: 436ns auth: 22.349094ms key: __NOT_EXISTS_KEY__zCN2Y99Zje get: 138.543µs del: 113.781µs
2020-04-20 11:36:51.011867497 +0800 CST m=+30.859398838 Time: connect: 544ns auth: 47.38697ms key: __NOT_EXISTS_KEY__xoIqpJfATT get: 184.15µs del: 136.089µs
2020-04-20 11:36:54.912074078 +0800 CST m=+34.759605426 Time: connect: 551ns auth: 75.628751ms key: __NOT_EXISTS_KEY__Rihq6fMg6o get: 142.938µs del: 117.792µs
2020-04-20 11:37:04.61194011 +0800 CST m=+44.459471447 Time: connect: 329ns auth: 37.372742ms key: __NOT_EXISTS_KEY__kml2BYmIWU get: 142.727µs del: 116.262µs
2020-04-20 11:37:08.336843054 +0800 CST m=+48.184374443 Time: connect: 455ns auth: 183.293µs key: __NOT_EXISTS_KEY__agEoV1XIEi get: 332.57µs del: 1.009885ms
2020-04-20 11:37:13.569478218 +0800 CST m=+53.417009565 Time: connect: 537ns auth: 208.941µs key: __NOT_EXISTS_KEY__elsn1cwmsp get: 123.57µs del: 113.602µs
2020-04-20 11:37:14.512036533 +0800 CST m=+54.359567881 Time: connect: 441ns auth: 37.097575ms key: __NOT_EXISTS_KEY__pDq4ybtCSs get: 131.325µs del: 145.883µs
2020-04-20 11:37:25.511796699 +0800 CST m=+65.359328038 Time: connect: 447ns auth: 28.796572ms key: __NOT_EXISTS_KEY__MMroOZD2IE get: 127.192µs del: 115.414µs
```

这是客户端直连一个后端redis的耗时统计，超过1ms抖动的部分。

- redis处理慢？

对应的auth命令长耗时，后端并没有对应的slowlog记录，理论上auth这种命令也不会出现在slowlog中。

- 祭出大杀器，tcpdump抓包

分别在客户端和服务进行抓包，分析看下对应长耗时到底在哪一部分。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/20200421233255.png)

上面是在服务端抓到的包，图中标黑的3个包就是`AUTH`对应的请求。

第一个包是收到客户端发来的`AUTH`包。

第二个是内核自动回复的一个delay ack包，可以看到delay了41ms。

第三个是应用层回复的`AUTH`包结果，这个回包又经过了3ms。

理论上，如果应用层能够及时返回信息，操作系统是没有必要单独发送一个delay ack空包，ack会携带在第三个包中发送给客户端。

很大可能的原因就是redis进程没有被操作调度，所以进程没有及时返回`AUTH`结果！！！

- k8s pod cpu limit?

![](https://raw.githubusercontent.com/yongman/i/img/picgo/20200421234115.png)

通过查看pod中`cat /sys/fs/cgroup/cpu,cpuaact/cpu.stat`查看统计信息，的确存在很多cpu throttle计数，说明的确cgroup限制了进程的正常调度。理论上在cpu很空闲的情况下，cpu时间片足够处理这么简单请求，不应该被throttle。

k8s中的[issue](https://github.com/kubernetes/kubernetes/issues/67577)信息中，提到这个是内核的一个bug，会导致limit限制调度准确，导致正常情况的进程被频繁重新调度。此bug在内核`linux-kernel 4.18`中已经修复。

- 去掉cgroup cpu limit限制

由于线上环境linux内核版本为`linux-kernel 4.9`的确存在该问题，直接将线上运行的pod cpu limit全部去掉，新创建的资源不再设置cpu limit。

请求抖动问题消失，总体耗时恢复正常水平。



**参考**

1. https://github.com/kubernetes/kubernetes/issues/67577

2. https://github.com/kubernetes/kubernetes/issues/51135

3. https://gist.github.com/bobrik/2030ff040fad360327a5fab7a09c4ff1

4. https://github.com/torvalds/linux/commit/512ac999d2755d2b7109e996a76b6fb8b888631d#diff-1c5364196d98130348bddabaad0a701f


