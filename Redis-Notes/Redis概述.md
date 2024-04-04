# Redis概述

### Redis是什么

- Redis（Remote Dictonary Server），即远程字典服务（通过远程网络的方式进行交互）
- Redis是内存中管道数据结构存储系统（内存数据库），可以用作==数据库==、==缓存==、==消息中间件MQ==
  - 关系型数据库主要将数据存在磁盘中，每次操作时需要将数据刷到磁盘中

### Redis能干嘛

- 内存存储，持久化（内存断电即失，rdb，aof）
- 效率高，可用于高速缓存
- 发布订阅系统
- 地图信息分析
- 计时器、计数器（浏览量）
- ......

### Redis特性

- 多样的数据类型
- 持久化
- 集群
- 事务
- ......

#### 

### Redis启动及测试

- 在虚拟机/usr/local/bin目录下
- 1、先以自己的配置文件启动server

```bash
redis-server kconfig/redis.conf
```

- 2、指定默认端口6379启动client，此时可以做各种get及set操作

```bash
redis-cli -p 6379
```

- 3、压力测试，100个并发客户端，每个并发100000个请求

```bash
redis-benchmark -h localhost -p 6379 -c 100 -n 100000
```



### Redis基本知识

- redis有16个数据库，默认使用第0个，可以通过select进行切换

```bash
select 3
```

- 通过DBSIZE查看数据库目前大小

```bash
DBSIZE
```

- 通过set往数据库里插入条目，以name为key，value为mcw

```bash
set name mcw
```

- 通过get以name为key从数据库中取得条目

```bash
get name
```

- 通过keys *可以查看数据库所有的key

```bash
keys *
```

- 通过flushall和flushdb清除数据库，flushall清空所有数据库，flushdb清空当前数据库

```bash
flushdb
```

- #### ==Redis是单线程的==

  - Redis是基于内存操作的，CPU不是Redis性能瓶颈
  - Redis的瓶颈是根据机器的内存和网络带宽决定的
  - 所以能使用单线程就使用单线程

- Redis单线程为什么还这么快

- 误区：

  - 高性能的服务器一定是多线程的？
  - 多线程（CPU调度，上下文切换）一定比单线程效率高？

- 首先了解：CPU速度 > 内存 > 硬盘

- 核心：Redis将所有的数据全部放在内存中，单线程效率是最高的

  - 多线程由于CPU上下文切换，更耗时
  - 对于Redis这样的内存系统来说，没有上下文切换效率就是最高的
  - 多次读写都是在一个CPU上的，在内存情况下，这个就是最佳的方案