# InnoDB引擎

# 逻辑存储结构

![image](https://github.com/tjumcw/notes/assets/106053649/2f74b506-8e6d-4acf-9754-c25098c5a0cd)


表空间（ibd文件）：

- 一个mysql实例可以对应多个表空间，用于存储记录、索引等数据

段：

- 分为数据段（Leaf node segment）、索引段（Non-leaf node segment）、回滚段（Rollback segment）
- InnoDB是索引组织表，数据段就是B+树的叶子节点， 索引段即为B+树的非叶子节点。
- 段用来管理多个Extent（区）

区：

- 每个区的大小为1M。 默认情况下， InnoDB存储引擎页大小为16K， 即一个区中一共有64个连续的页

页：

- 是InnoDB 存储引擎磁盘管理的最小单元，每个页的大小默认为 16KB。
- 为了保证页的连续性，InnoDB 存储引擎每次从磁盘申请 4-5 个区。

行：

- 行，InnoDB 存储引擎数据是按行进行存放的。

>在行中，默认有两个隐藏字段：
>
>- Trx_id：每次对某条记录进行改动时，都会把对应的事务id赋值给trx_id隐藏列。
>
>- Roll_pointer：每次对某条引记录进行改动时，都会把旧的版本写入到undo日志中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息。



# 架构

## 内存结构

### Buffer Pool

- InnoDB存储引擎基于磁盘文件存储，访问物理硬盘和在内存中进行访问，速度相差很大。

- 为了尽可能弥补这两者之间的I/O效率的差值，就需要把经常使用的数据加载到缓冲池中，避免每次访问都进行磁盘I/O。

- 缓冲池 Buffer Pool，是主内存中的一个区域，里面可以缓存磁盘上经常操作的真实数据。
- 在执行增删改查操作时，先操作缓冲池中的数据（若缓冲池没有数据，则从磁盘加载并缓存）
- 然后再以一定频率刷新到磁盘，从而减少磁盘IO，加快处理速度。

缓冲池以Page页为单位，底层采用链表数据结构管理Page。根据状态，将Page分为三种类型：

- free page：空闲page，未被使用。

- clean page：被使用page，数据没有被修改过。

- dirty page：脏页，被使用page，数据被修改过，也中数据与磁盘的数据产生了不一致。



### Change Buffer

Change Buffer，更改缓冲区（针对于非唯一二级索引页）

- 在执行DML语句时，如果这些数据Page没有在Buffer Pool中，不会直接操作磁盘，而会将数据变更存在更改缓冲区 Change Buffer中

- 在未来数据被读取时，再将数据合并恢复到Buffer Pool中，再将合并后的数据刷新到磁盘中。

>与聚集索引不同，二级索引通常是非唯一的，并且以相对随机的顺序插入二级索引。同样，删除和更新可能会影响索引树中不相邻的二级索引页，如果每一次都操作磁盘，会造成大量的磁盘IO。有了ChangeBuffer之后，我们可以在缓冲池中进行合并处理，减少磁盘IO。



###  Adaptive Hash Index

- 自适应hash索引，用于优化对Buffer Pool数据的查询。

- MySQL的innoDB引擎中虽然没有直接支持hash索引，但是给我们提供了一个功能就是这个自适应hash索引。

>InnoDB存储引擎会监控对表上各索引页的查询，如果观察到在特定的条件下hash索引可以提升速度，则建立hash索引，称之为自适应hash索引。



### Log Buffer

- 日志缓冲区，用来保存要写入到磁盘中的log日志数据（redo log 、undo log）





## 磁盘结构

主要存储undolog和redolog





## 后台线程

Master Thread：

- 核心后台线程，负责调度其他线程，还负责将缓冲池中的数据异步刷新到磁盘中, 保持数据的一致性，还包括脏页的刷新、合并插入缓存、undo页的回收

IO Thread：

- 在InnoDB存储引擎中大量使用了AIO来处理IO请求, 这样可以极大地提高数据库的性能，而IOThread主要负责这些IO请求的回调

| **线程类型**         | **默认个数** | **职责**                     |
| -------------------- | ------------ | ---------------------------- |
| Read thread          | 4            | 负责读操作                   |
| Write thread         | 4            | 负责写操作                   |
| Log thread           | 1            | 负责将日志缓冲区刷新到磁盘   |
| Insert buffer thread | 1            | 负责将写缓冲区内容刷新到磁盘 |

Purge Thread：

- 主要用于回收事务已经提交了的undo log，在事务提交之后，undo log可能不用了，就用它来回收

Page Cleaner Thread：

- 协助 Master Thread 刷新脏页到磁盘的线程，它可以减轻 Master Thread 的工作压力，减少阻塞





# 事务

## 事务基础

事务的原子性、一致性和持久性，主要通过redo log和undo log实现

事务的隔离性，主要通过锁 + MVCC实现



## redo log

### 基本概念

> 重做日志，记录的是事务提交时数据页的物理修改，是用来实现事务的持久性。

该日志文件由两部分组成：

- 重做日志缓冲（redo log buffer）以及重做日志文件（redo logfile）

- 前者是在内存中，后者在磁盘中。
- 当事务提交之后会把所有修改信息都存到该日志文件中, 用于在刷新脏页到磁盘,发生错误时, 进行数据恢复使用。



### 无redo log

- 在InnoDB引擎中的内存结构中，主要的内存区域就是缓冲池，在缓冲池中缓存了很多的数据页。

- 当我们在一个事务中，执行多个增删改的操作时，InnoDB引擎会先操作缓冲池中的数据。

- 如果缓冲区没有对应的数据，会通过后台线程将磁盘中的数据加载出来，存放在缓冲区中。

- 然后将缓冲池中的数据修改，修改后的数据页我们称为脏页。 

- 而脏页则会在一定的时机，通过后台线程刷新到磁盘中，从而保证缓冲区与磁盘的数据一致。

-  而缓冲区的脏页数据并不是实时刷新的，而是一段时间之后将缓冲区的数据刷新到磁盘中。

- 假如刷新到磁盘的过程出错了，而提示给用户事务提交成功，而数据却没有持久化下来，这就出现问题了，没有保证事务的持久性。



### 有redo log

- 有了redolog之后，当对缓冲区的数据进行增删改之后，会首先将操作的数据页的变化，记录在redolog buffer中。
- 在事务提交时，会将redo log buffer中的数据刷新到redo log磁盘文件中。
- 过一段时间之后，如果刷新缓冲区的脏页到磁盘时，发生错误，此时就可以借助于redo log进行数据恢复，这样就保证了事务的持久性。
-  而如果脏页成功刷新到磁盘 或 或者涉及到的数据已经落盘，此时redolog就没有作用了，就可以删除了，所以存在的两个redolog文件是循环写的。



## undo log

> 回滚日志，用于记录数据被修改前的信息 , 作用包含两个 : 提供回滚(保证事务的原子性) 和MVCC(多版本并发控制) 。

- undo log和redo log记录物理日志不一样，它是逻辑日志。

- 可以认为当delete一条记录时，undolog中会记录一条对应的insert记录。

- 反之亦然，当update一条记录时，它记录一条对应相反的update记录。

- 当执行rollback时，就可以从undo log中的逻辑记录读取到相应的内容并进行回滚。

Undo log销毁：undo log在事务执行时产生，事务提交时，并不会立即删除undo log，因为这些日志可能还用于MVCC。

Undo log存储：undo log采用段的方式进行管理和记录，存放在前面介绍的 rollback segment回滚段中，内部包含1024个undo log segment。





# MVCC

## 基本概念

### 当前读

> 读取的是记录的最新版本，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加锁

> 对于我们日常的操作，如：select ... lock in share mode(共享锁)，select ... for update、update、insert、delete(排他锁)都是一种当前读。

示例：

- 在事务A中执行以下操作

```sql
begin;
select * from user;
+----+--------+------+
| id | name   | age  |
+----+--------+------+
|  1 | Cpp    |   21 |
|  5 | Java   |   10 |
|  6 | Php    |   26 |
|  9 | Python |   29 |
+----+--------+------+
```

- 在事务B中执行以下操作

```sql
begin;
update user set age = 33 where id = 1;
```

- 在事务A中执行相同查询语句，没变化（事务B没提交）

```sql
select * from user;
+----+--------+------+
| id | name   | age  |
+----+--------+------+
|  1 | Cpp    |   21 |
|  5 | Java   |   10 |
|  6 | Php    |   26 |
|  9 | Python |   29 |
+----+--------+------+
```

- 事务B执行commit提交之后，事务A执行同样SQL

```sql
select * from user;
+----+--------+------+
| id | name   | age  |
+----+--------+------+
|  1 | Cpp    |   21 |
|  5 | Java   |   10 |
|  6 | Php    |   26 |
|  9 | Python |   29 |
+----+--------+------+
```

>因为事务隔离级别是可重复读，所以依旧没变化

- 如果执行以下SQL，获得的就是当前读，哪怕是在事务A没提交也可以获取最新数据

```sql
select * from user lock in share mode ;
+----+--------+------+
| id | name   | age  |
+----+--------+------+
|  1 | Cpp    |   33 |
|  5 | Java   |   10 |
|  6 | Php    |   26 |
|  9 | Python |   29 |
+----+--------+------+
```



### 快照读

> 简单的select（不加锁）就是快照读，快照读，读取的是记录数据的可见版本，有可能是历史数据，不加锁，是非阻塞读。

- Read Committed：每次select，都生成一个快照读。

- Repeatable Read：开启事务后第一个select语句才是快照读的地方。
  - 后续的相同查询直接查的就是这个快照

- Serializable：快照读会退化为当前读。



### MVCC

- 全称 Multi-Version Concurrency Control，多版本并发控制。

- 指维护一个数据的多个版本，使得读写操作没有冲突，快照读为MySQL实现MVCC提供了一个非阻塞读功能。

- MVCC的具体实现，还需要依赖于数据库记录中的三个隐式字段、undo log日志、readView。



## 隐藏字段

| **隐藏字段** | **含义**                                                     |
| ------------ | ------------------------------------------------------------ |
| DB_TRX_ID    | 最近修改事务ID，记录插入这条记录或最后一次修改该记录的事务ID。 |
| DB_ROLL_PTR  | 回滚指针，指向这条记录的上一个版本，用于配合undo log，指向上一个版本。 |
| DB_ROW_ID    | 隐藏主键，如果表结构没有指定主键，将会生成该隐藏字段。       |



## undo log

### 介绍

回滚日志，在insert、update、delete的时候产生的便于数据回滚的日志。

- 当insert的时候，产生的undo log日志只在回滚时需要，在事务提交后，可被立即删除。

- 而update、delete的时候，产生的undo log日志不仅在回滚时需要，在快照读时也需要，不会立即被删除。



### 版本链

有一张原始表数据为

![image-20240405214537467](C:\Users\HASEE\AppData\Roaming\Typora\typora-user-images\image-20240405214537467.png)

>DB_TRX_ID : 代表最近修改事务ID，记录插入这条记录或最后一次修改该记录的事务ID，是自增的。
>
>DB_ROLL_PTR ： 由于这条数据是才插入的，没有被更新过，所以该字段值为null。

然后，有四个并发事务同时在访问这张表



#### 第一步

![image-20240405214628004](C:\Users\HASEE\AppData\Roaming\Typora\typora-user-images\image-20240405214628004.png)

>当事务2执行第一条修改语句时，会记录undo log日志，记录数据变更之前的样子; 
>
>然后更新记录，并且记录本次操作的事务ID，回滚指针，回滚指针用来指定如果发生回滚，回滚到哪一个版本。

![image-20240405214703002](C:\Users\HASEE\AppData\Roaming\Typora\typora-user-images\image-20240405214703002.png)



#### 第二步

![image-20240405214744607](C:\Users\HASEE\AppData\Roaming\Typora\typora-user-images\image-20240405214744607.png)

>当事务3执行第一条修改语句时，也会记录undo log日志，记录数据变更之前的样子; 
>
>然后更新记录，并且记录本次操作的事务ID，回滚指针，回滚指针用来指定如果发生回滚，回滚到哪一个版本。

![image-20240405214808962](C:\Users\HASEE\AppData\Roaming\Typora\typora-user-images\image-20240405214808962.png)



#### 第三步

![image-20240405214831271](C:\Users\HASEE\AppData\Roaming\Typora\typora-user-images\image-20240405214831271.png)

>当事务4执行第一条修改语句时，也会记录undo log日志，记录数据变更之前的样子; 
>
>然后更新记录，并且记录本次操作的事务ID，回滚指针，回滚指针用来指定如果发生回滚，回滚到哪一个版本。

![image-20240405214852627](C:\Users\HASEE\AppData\Roaming\Typora\typora-user-images\image-20240405214852627.png)

>最终我们发现，不同事务或相同事务对同一条记录进行修改，会导致该记录的undolog生成一条记录版本链表
>
>链表的头部是最新的旧记录，链表尾部是最早的旧记录。





## readview

> ReadView（读视图）是 快照读 SQL执行时MVCC提取数据的依据，记录并维护系统当前活跃的事务（未提交的）id。

ReadView中包含了四个核心字段：

| **字段**       | **含义**                                             |
| -------------- | ---------------------------------------------------- |
| m_ids          | 当前活跃的事务ID集合                                 |
| min_trx_id     | 最小活跃事务ID                                       |
| max_trx_id     | 预分配事务ID，当前最大事务ID+1（因为事务ID是自增的） |
| creator_trx_id | ReadView创建者的事务ID                               |

而在readview中就规定了快照读对于版本链数据的访问规则：

> trx_id 代表当前undolog版本链对应事务ID

| **条件**                           | **是否可以访问**                          | **说明**                                   |
| ---------------------------------- | ----------------------------------------- | ------------------------------------------ |
| trx_id == creator_trx_id           | 可以访问该版本                            | 成立，说明数据是当前这个事务更改的。       |
| trx_id < min_trx_id                | 可以访问该版本                            | 成立，说明数据已经提交了。                 |
| trx_id > max_trx_id                | 不可以访问该版本                          | 成立，说明该事务是在ReadView生成后才开启。 |
| min_trx_id <= trx_id <= max_trx_id | 如果trx_id不在m_ids中，是可以访问该版本的 | 成立，说明数据已经提交。                   |



不同的隔离级别，生成ReadView的时机不同：

- READ COMMITTED ：在事务中每一次执行快照读时生成ReadView。

- REPEATABLE READ：仅在事务中第一次执行快照读时生成ReadView，后续复用该ReadView。



## 原理分析

### 读已提交（RC）级别

> RC隔离级别下，在事务中每一次执行快照读时生成ReadView。

分析事务5中，两次快照读读取数据，是如何获取数据的?

- 在事务5中，查询了两次id为30的记录，由于隔离级别为Read Committed
- 所以每一次进行快照读都会生成一个ReadView，那么两次生成的ReadView如下。

![image-20240405215931039](C:\Users\HASEE\AppData\Roaming\Typora\typora-user-images\image-20240405215931039.png)

- 对应的undo log版本链及readview规则如下（针对第一次查询readview）

![image-20240405220217339](C:\Users\HASEE\AppData\Roaming\Typora\typora-user-images\image-20240405220217339.png)

>- 在进行匹配时，会从undo log的版本链，从上到下进行挨个匹配
>
>- 也就是说会先拿最新的老记录，DB_TRX_ID为4的套右边的规则
>  - 4不满足第1，2，3条，由于4在m_ids中，也不满足第4条
>
>- 继续沿着链找3的套右边规则，也不满足，继续沿着链找
>- 发现DB_TRX_ID为2的满足第2条规则，即这个版本的记录九尾所查找的快照读
>- 第二次查询的readview也类似分析即可



### 可重复读（RR）级别

> RR隔离级别下，仅在事务中第一次执行快照读时生成ReadView，后续复用该ReadView。 
>
> 而RR 是可重复读，在一个事务中，执行两次相同的select语句，查询到的结果是一样的。

![image-20240405220938399](C:\Users\HASEE\AppData\Roaming\Typora\typora-user-images\image-20240405220938399.png)

>我们看到，在RR隔离级别下，只是在事务中第一次快照读时生成ReadView，后续都是复用该ReadView。
>
>那么既然ReadView都一样， ReadView的版本链匹配规则也一样， 那么最终快照读返回的结果也是一样的。
>
>分析的流程和规则也是和上述分析一摸一样，根据undo log版本链 + readview规则比对即可得出对应的快照读版本。