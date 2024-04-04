# Redis主从复制（类似Raft）

### 概念

![image](https://user-images.githubusercontent.com/106053649/181399039-87873c6f-69c2-4e86-8ffe-fd4803e907f2.png)

- 主从复制和读写分离，一个主机多个从机，主机负责写，从机负责读（大部分的操作都是读操作）
- 主从复制的用处
  - 结构上，单个Redis处理所有请求负载压力较大，且单个Redis发生单点故障就都没了
  - 容量上，单个Redis服务器内存容量有限。哪怕服务器Redis内存为256G，也不可能将所有内存用作Redis存储内存，一般来说，单台Redis最大使用内存不应该超过20G
  - 应用上，大多是“多读少写”，一个主机多个从机更合理

- 默认情况下，每个Redis服务器都是主机，且一个主机能有多个从机，但一个从机只能有一个主机



### 架构

![image](https://user-images.githubusercontent.com/106053649/181399072-5b5c005a-b36c-45b8-82cb-a1e1ae227776.png)



### 环境配置

只需配置从库，不用配置主库

```bash
127.0.0.1:6379> info replication		#查看当前库的信息
# Replication
role:master
connected_slaves:0
master_replid:bc2ec8407cef0c4c12d40041ac94ad913458be26
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0
second_repl_offset:-1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```



复制三个配置文件，修改对应的信息

1、端口号

2、pid名字

3、log文件名字

4、dump.rdb文件名字



- 开启三个服务，通过进程信息查看

```bash
zsf@zsf:~$ ps -aux | grep redis
root       455  0.1  0.0  55972  4564 ?        Ssl  13:44   0:00 redis-server 127.0.0.1:6379
root       472  0.1  0.0  55972  4536 ?        Ssl  13:45   0:00 redis-server 127.0.0.1:6380
root       489  0.0  0.0  55972  4372 ?        Ssl  13:45   0:00 redis-server 127.0.0.1:6381
zsf        494  0.0  0.0  16180  1108 pts/9    S+   13:45   0:00 grep --color=auto redis
```



### 一主二从

- ==默认情况下，每个Redis服务器都是主节点==，一般情况下只需要配置从机即可

- 通过SLAVEOF host port 命令配置，选择一个host的端口为主机

```bash
127.0.0.1:6380> SLAVEOF 127.0.0.1 6379
OK
127.0.0.1:6380> info replication
# Replication
role:slave							#当前角色是从机
master_host:127.0.0.1				#可以查看主机的信息
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_repl_offset:28
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:e396125efa7605d96baa638feb35122ac4d46351
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:28
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:29
repl_backlog_histlen:0

#都配置完后，在主机中查看
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6381,state=online,offset=196,lag=1
slave1:ip=127.0.0.1,port=6380,state=online,offset=196,lag=0
master_replid:e396125efa7605d96baa638feb35122ac4d46351
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:196
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:196
```

- 真实的配置应该在配置文件中修改（永久的），通过命令配置是暂时的



#### 细节及测试

- 主机可以写，从机不能写只能读。主机中的所有信息和数据，都会被从机自动保存



主机写：

```bash
127.0.0.1:6379> set k1 v1
OK
```



从机只能读：

```bash
127.0.0.1:6381> get k1
"v1"
127.0.0.1:6381> set k2 v2
(error) READONLY You can't write against a read only replica.
```



让主机强制断了，从机仍然是从机

```bash
#让主机强制宕机
127.0.0.1:6379> SHUTDOWN
not connected> exit

#从机身份不变
127.0.0.1:6380> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:down
master_last_io_seconds_ago:-1
master_sync_in_progress:0
slave_repl_offset:710
master_link_down_since_seconds:69
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:e396125efa7605d96baa638feb35122ac4d46351
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:710
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:29
repl_backlog_histlen:682

#从机依旧能获取主机之前写入的数据
127.0.0.1:6380> get k1
"v1"
```



让主机恢复上线，主机依旧可以获取主机新写入的信息

```bash
#主机恢复上线，并set k2
127.0.0.1:6379> set k2 v2
OK

#从机依旧能读取主机新写入的数据
127.0.0.1:6380> get k2
"v2"
```



让其中一个从机下线（6381），主机set k3，再重新让6381上线

```bash
#在从机6381下线的情况下，主机set k3
127.0.0.1:6379> set k3 v3
OK

#原先的从机6381重新上线，去get k3
127.0.0.1:6381> get k3
(nil)

#原先的从机6381上线后，主机set k4
127.0.0.1:6379> set k4 v4
OK

#此时服务器6381去get k4
127.0.0.1:6381> get k4
(nil)

#但是6381下线前从主机复制的数据还在
127.0.0.1:6381> keys *
1) "k1"
2) "k2"

#断线后恢复，默认又成了主机（没修改配置文件，只是从命令行设置的主从关系）
127.0.0.1:6381> info replication
# Replication
role:master
connected_slaves:0
master_replid:abdd91eb880ac4c839a509898315d5a5d858a506
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0
second_repl_offset:-1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```

- 如果使用命令行配置的主从，如果重启了就会变回主机，但只要变回从机，会立刻从主机中获取值



此时若通过SLAVEOF命令让服务器6381重新变回6379的从机，再去get k4

```bash
#重新让6381成为6379的主机
127.0.0.1:6381> SLAVEOF 127.0.0.1 6379
OK
127.0.0.1:6381> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
master_last_io_seconds_ago:2
master_sync_in_progress:0
slave_repl_offset:950
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:d5d7abefdb9f488c05de6c8414913e8636f9ef69
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:950
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:937
repl_backlog_histlen:14

#再去get k4
127.0.0.1:6381> get k4
"v4"
```



#### 复制原理

- 针对上述的测试，有以下的两种复制原理
  - 全量复制
  - 增量复制

![image](https://user-images.githubusercontent.com/106053649/181399137-0e1b25c2-45d6-495c-8a59-d518958f0055.png)



### 另一种主从复制关系（层层链路）

- 6739是6380的主机，6380是6381的主机
- 即6379负责写，将数据传给6380，6380将复制来的新增数据传给6381



- 6381配置后的信息

```bash
127.0.0.1:6381> SLAVEOF 127.0.0.1 6380
OK
127.0.0.1:6381> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6380
master_link_status:up
master_last_io_seconds_ago:6
master_sync_in_progress:0
slave_repl_offset:1916
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:d5d7abefdb9f488c05de6c8414913e8636f9ef69
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:1916
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:937
repl_backlog_histlen:980
```



- 配置完6381后6379的信息，发现只有一个从机了

```bash
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.1,port=6380,state=online,offset=2000,lag=0
master_replid:d5d7abefdb9f488c05de6c8414913e8636f9ef69
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:2000
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:2000
```



- 再看最特殊的6380的配置信息（依旧是从节点，不能写入）

```bash
127.0.0.1:6380> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
master_last_io_seconds_ago:6
master_sync_in_progress:0
slave_repl_offset:2070
slave_priority:100
slave_read_only:1
connected_slaves:1
slave0:ip=127.0.0.1,port=6381,state=online,offset=2070,lag=1	#但多了一个从机的信息
master_replid:d5d7abefdb9f488c05de6c8414913e8636f9ef69
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:2070
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:2070
```



- 此时在主机6379中set k5

```bash
127.0.0.1:6379> set k5 v5
OK
```

- 从机6380中去get k5

```bash
127.0.0.1:6380> get k5
"v5"
```

- 6380的从机6381get k5

```bash
127.0.0.1:6381> get k5
"v5"
```



- 此时若是6379宕机，6380和6381都是从机，其中6381是6380的从机

- 在不使用哨兵模式的情况下，如果主机没法恢复了，需要手动配置将从机改为主机（谋朝篡位）
  - 通过SLAVEIF no one命令让自己变为主机，其他的节点就可以==手动==来连接到这个主机
- 若在从机6380下执行该命令，查看其信息如下（角色变为主机）：

```bash
127.0.0.1:6380> SLAVEOF no one
OK
127.0.0.1:6380> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.1,port=6381,state=online,offset=2710,lag=0
master_replid:39be3da30b20ebdef99170fec228f50f6df3e3ae
master_replid2:d5d7abefdb9f488c05de6c8414913e8636f9ef69
master_repl_offset:2710
second_repl_offset:2711
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:2710
```

- 此时哪怕6379重新上线，6380仍旧是主机（只能重新配置了）



### ==哨兵模式==

#### 概念

能够后台监控主机是否故障，如果故障了==根据投票数自动将从机转换为主机==

- 哨兵模式是一种特殊的模式，首先Redis提供了哨兵的命令。哨兵是一个独立的进程，作为进程它会独立运行。
- 原理是==哨兵通过发送命令，等待Redis服务器响应，从而监控运行的多个Redis实例==

![image](https://user-images.githubusercontent.com/106053649/181399176-dc643d77-ec54-4080-b2bc-53b0ad9fc8de.png)

这里的哨兵有两个作用

- 通过发送命令，让Redis服务器返回监控其运行状态，包括主服务器和从服务器
- 当哨兵监测到master宕机，会自动将slave切换成master，然后通过==发布订阅模式==通知其他的从服务器，修改配置文件，让他们切换主机



考虑到一个哨兵对Redis服务器进行监控，可能会出现问题，考虑哨兵也形成集群，即多哨兵模式

- 利用多个哨兵进行监控Redis服务器
- 各个哨兵之间还会互相监控

![image](https://user-images.githubusercontent.com/106053649/181399197-c657a1d2-d980-4a08-817c-64c5324a633b.png)



#### 测试

目前的状态是一主二从（6379主机，挂这两个从机6380和6381）

- 配置哨兵配置文件 sentinel.conf

```bash
#sentinel monitor 被监控的名称（不关键） host port 1
sentinel monitor myredis 127.0.0.1 6379 1
```

- 后面这个数字1，表示主机挂了，slave投票看让谁成为主机



- 启动哨兵

```bash
redis-sentinel kconfig/sentinel.conf
```

![image](https://user-images.githubusercontent.com/106053649/181399221-a328b2cd-e3fa-41d6-97f9-5bb13415ef50.png)



- 查看主机当前所有keys，并让主机下线

```bash
127.0.0.1:6379> keys *
1) "k5"
2) "k4"
3) "k3"
4) "k1"
5) "k2"
127.0.0.1:6379> SHUTDOWN
not connected> exit
```



- 刚下线时6380和6381依旧是从机，过一会后哨兵监测到6379下线会发起重新投票选举

![image](https://user-images.githubusercontent.com/106053649/181399243-1e4a6960-e702-40bc-a909-708def99e075.png)



- 此时去查看服务器信息，发现6380成了master

```bash
127.0.0.1:6380> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=127.0.0.1,port=6381,state=online,offset=19484,lag=1
master_replid:d42be3a9d7cba0f19b1175b5ea9283e972fcc577
master_replid2:d5d7abefdb9f488c05de6c8414913e8636f9ef69
master_repl_offset:19616
second_repl_offset:14202
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2711
repl_backlog_histlen:16906
```



- 此时主机在连接回来也没用，反而会成为6380的从机

```bash
9163:X 27 Jul 2022 15:13:17.187 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ myredis 127.0.0.1 6380
9163:X 27 Jul 2022 15:13:47.216 # +sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ myredis 127.0.0.1 6380
9163:X 27 Jul 2022 15:16:50.257 # -sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ myredis 127.0.0.1 6380
9163:X 27 Jul 2022 15:17:00.270 * +convert-to-slave slave 127.0.0.1:6379 127.0.0.1 6379 @ myredis 127.0.0.1 6380
```

- 此时查看配置信息发现6379已成为6380的从机了

```bash
127.0.0.1:6379> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6380
master_link_status:up
master_last_io_seconds_ago:2
master_sync_in_progress:0
slave_repl_offset:33939
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:d42be3a9d7cba0f19b1175b5ea9283e972fcc577
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:33939
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:29199
repl_backlog_histlen:4741
```



#### 哨兵模式优缺点

优点：

- 哨兵集群，基于主从复制模式，所有的主从配置优点，它全有
- 主从可以切换，==故障可以转移==，系统的可用性会更好
- 哨兵模式就是主从模式的升级，手动到自动，更加健壮

缺点：

- Redis不好在线扩容，集群容量一旦达到上限，在线扩容就十分麻烦
- 实现哨兵模式的配置比较麻烦
