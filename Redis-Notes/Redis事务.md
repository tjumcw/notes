# Redis事务

### Redis概念

- 本质：一组命令的集合
- 特性：一次性、顺序性、排他性
- 将一组命令插入队列，所有命令会被序列化，按照顺序执行

- Redis单条命令保证原子性，但是事务不保证原子性
- Redis事务没有隔离级别的概念
- Redis的命令在事务中，并没有直接被执行，只有发起执行命令的时候才会执行（Exec）

- Redis的事务
  - 开启事务（multi）
  - 命令入队（...）
  - 执行事务（exec）



### 正常执行事务

```bash
127.0.0.1:6379> multi		#开启事务
OK
127.0.0.1:6379> set k1 v1
QUEUED						#命令入队
127.0.0.1:6379> set k2 v2
QUEUED
127.0.0.1:6379> get k2
QUEUED
127.0.0.1:6379> set k3 v3
QUEUED
127.0.0.1:6379> exec		#执行事务
1) OK
2) OK
3) "v2"
4) OK
```



### 放弃事务（DISCARD）

```bash
127.0.0.1:6379> multi		#开启事务
OK
127.0.0.1:6379> set k1 v1
QUEUED
127.0.0.1:6379> set k2 v2
QUEUED
127.0.0.1:6379> set k4 v4
QUEUED
127.0.0.1:6379> DISCARD		#取消事务
OK
127.0.0.1:6379> get k4		#事务队列中命令都不会被执行
(nil)
```



### 编译型异常（代码有问题）

- 命令有错，事务中的所有命令都不会被执行

```bash
127.0.0.1:6379> set k1 v1
QUEUED
127.0.0.1:6379> set k2 v2
QUEUED
127.0.0.1:6379> set k3 v3
QUEUED
127.0.0.1:6379> getset k3		#错误的命令
(error) ERR wrong number of arguments for 'getset' command
127.0.0.1:6379> set k4 v4
QUEUED
127.0.0.1:6379> set k5 v5
QUEUED
127.0.0.1:6379> EXEC			#执行事务出错
(error) EXECABORT Transaction discarded because of previous errors.
127.0.0.1:6379> get k1			#所有的命令都不会被执行
(nil)
```



### 运行时异常

- 如果事务队列中命令存在语法错误运行时才能检测出来，那么执行命令时，其他命令可以正常执行，错误命令抛出异常
- 也就是说Redis事务是不保证原子性的（单条命令保证原子性）

```bash
127.0.0.1:6379> set k1 "v1"
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> incr k1		#会执行失败
QUEUED
127.0.0.1:6379> set k2 v2
QUEUED
127.0.0.1:6379> get k2
QUEUED
127.0.0.1:6379> EXEC		#执行事务报错
1) (error) ERR value is not an integer or out of range	#虽然第一条命令报错了，但事务依旧正常执行了
2) OK
3) "v2"
```



### 监控（Watch）-> 充当乐观锁（面试常问）

#### 悲观锁：

- 很悲观，什么时候都觉得出问题，无论做什么都会加锁（性能受影响）

#### 乐观锁

- 很乐观，认为什么时候都不会出问题，所以不会上锁，在更新数据时判断：在此期间是否有人修改过该数据

- MySQL中通过获取version，在更新时比较version来实现乐观锁操作



### Redis监控测试

#### 正常执行成功

```bash
127.0.0.1:6379> set money 100
OK
127.0.0.1:6379> set out 0
OK
127.0.0.1:6379> WATCH money			#监视money对象
OK
127.0.0.1:6379> multi				#事务正常结束，数据期间没有发生变动，这个时候就正常执行成功
OK
127.0.0.1:6379> DECRBY money 20
QUEUED
127.0.0.1:6379> INCRBY out 20
QUEUED
127.0.0.1:6379> exec
1) (integer) 80
2) (integer) 20
```



#### 失败测试

- 多线程（另一个客户端）修改了监视的值，事务执行失败
- 使用watch可以当作redis的乐观锁操作

```bash
127.0.0.1:6379> watch money
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> DECRBY money 10
QUEUED
127.0.0.1:6379> incrby out 10
QUEUED
127.0.0.1:6379> exec			#执行之前，另外一个线程修改了money，导致事务执行失败
(nil)
```



- 如果事务执行失败了，先解锁（UNWATCH），再次获取锁（获取最新的值并再次监视）

```bash
127.0.0.1:6379> unwatch
OK
127.0.0.1:6379> watch money
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> decrby money 10
QUEUED
127.0.0.1:6379> incrby out 10
QUEUED
127.0.0.1:6379> exec
1) (integer) 990
2) (integer) 30
```

