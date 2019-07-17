---
title: 使用docker-compose一键运行tidis
categories:
  - [分布式]
  - [存储]
date: 2018-08-31 15:26:19
---

[Tidis](https://github.com/yongman/tidis)是基于tikv的兼容redis协议持久化存储，得益于分布式kv存储tikv，tidis能够实现水平扩展，数据安全存储和分布式事务支持。

1. 克隆tidis-docker-compose仓库

```
~ git clone https://github.com/yongman/tidis-docker-compose.git
```

2. 执行docker-compose

```
~ cd tidis-docker-compose/

~ docker-compose up -d
Creating network "docker_default" with the default driver
Creating docker_pd_1 ... done
Creating docker_tikv_1 ... done
Creating docker_tidis_1 ... done
```

3. 验证

```
redis-cli -p 5379
```

4. 停止并删除实例

```
~ docker-compose down
Stopping docker_tidis_1 ... done
Stopping docker_tikv_1  ... done
Stopping docker_pd_1    ... done
Removing docker_tidis_1 ... done
Removing docker_tikv_1  ... done
Removing docker_pd_1    ... done
Removing network docker_default
```
