# InnoDB引擎

# 逻辑存储结构

![image-20240405204018363](C:\Users\HASEE\AppData\Roaming\Typora\typora-user-images\image-20240405204018363.png)



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