---
title: 数据仓库Snowflake论文
categories: [存储]
date: 2020-01-15 17:53:38
---
数据仓库和数据库的区别就是：

- 数据库：OLTP，数据操作会包含读写，并且要求时延较低，操作结果会是决定上层操作成功与否的关键。数据量一般有限。
- 数据仓库或数据湖：OLAP，数据操作以读为主，极少update操作，对时延不是很敏感，一般查询比较复杂并且数据量大。

大数据场景下，Hadoop或Spark是社区比较活跃的解决方案，但是snowflake认为，它们并不高效，并且系统的维护和使用成本也很高。

分布式存储存储经典的两种架构：

- shared-disk：数据存储到相同位置，大家用同样的资源，Oracal Exadata，延展性和并发性不行。
- shared-nothing：近年来的主流做法，系统通过策略将资源分摊到多个结点，每个节点之间的数据不共享，但是实现上可能会用来保存其他节点的高可用副本数据，对于高并发或者大量的查询工作负载会分发到各个节点进行执行，所以延展性和并发性不受限制。

但是snowflake的架构图如下，在Shared-nothing基础上提出的Multi-Cluster, Shared Data Architecture，完全实现了计算存储分离。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/20200116163848.png)

1. 架构的最上层是服务化组件，包括查询优化器、元数据存储、鉴权、资源管理和事务管理等。

- 查询优化器：实现查询的管理和优化，将对应的查询计划分发Virtual Warehouse中特定的计算节点，为了提高cache效率和性能，优化器会对其中的节点进行类似一致性hash的管理。
- 事务管理：采用SI和MVCC进行的并发控制，整体的存储思想类似于LSM，数据不会原地修改，数据只能读取修改后整块写入，历史版本可以保留，SI的实现也是基于MVCC来实现的。

2. 中间是计算层Virtual Warehouse

本质就是vm计算资源，多个vm虚拟机资源加速本地的磁盘做cache就构成了逻辑上的一个虚拟的warehouse，其中做了cache命中率优化和类似的文件p2p分发，减轻存储层访问压力，提升性能。

最下层是存储层

3. 存储层

存储层使用的云上的对象存储，aws的s3，azure的blob和google的cloud storage。对象存储系统的接口很简单，基本上都是GET/DELETE/PUT三种，s3支持部分数据获取。snowflake数据chunk的存储也是采用range-based来存储，元数据依赖cloud service保存。



数据的安全性完全交给底层的对象存储，计算层不需要关心数据高可用和水平扩展能力。

snowflake提供了Pure-saas用户体验，与传统的DBaas相比，用户完全不用关心数据库高可用、数据库调优等。



论文中还对snowflake是如何对数据进行加密来保证数据安全性的做了大量工作，在我看来，还是比较吃惊的，他们对数据的安全性做了这么复杂的设计，而不仅仅是一个固定的密钥进行加密。其中包括key rotation、rekeying，并且传输的过程也实现端到端加密。

![](https://raw.githubusercontent.com/yongman/i/img/picgo/20200116172045.png)