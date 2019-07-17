---
title: 正向代理和反向代理
categories: [技术记录]
date: 2018-08-06 11:29:59
---

虽然经常使用nginx或者科学上网搭建代理，对于代理分类具体含义还是需要解析下。

## 正向代理

正向代理就是一个位于客户端和目标服务器的代理服务器，对于客户端不透明，如果使用正向代理，客户端一般需要经过一些配置，就像我们科学上网，一般客户端会经过配置，直接访问代理服务器，通过代理服务器来访问目标资源。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/dunEUb7.png)

## 反向代理

反向代理就是目标服务器，客户端任务它就是目标服务器，就像nginx的proxy_pass配置，用户访问时访问反向代理服务器，并不知道反向代理后面的资源在哪里。这种方式对用户透明，客户端不需要修改配置。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/pCN6p8i.png)

## 区别

- 用途

正向代理：典型就是应用在跨越防火墙来访问互联网，（自己在防火墙内，访问不了互联网）。
反向代理：典型就是将防火墙后面的计算资源提供给用户，（后端服务在防火墙内，用户访问不了，但是可以通过一个网关或者代理入口，如nginx）。

- 安全性

正向代理：允许客户端访问目标端并且隐藏自身的存在，需要一些必要鉴权措施保证自身的安全。
反向代理：对用户透明。

## 参考

1. [Forward Proxy vs Reverse Proxy](https://www.jscape.com/blog/bid/87783/Forward-Proxy-vs-Reverse-Proxy)
