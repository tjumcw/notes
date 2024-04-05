# SQL优化



# 插入数据

## insert优化

- 批量插入数据

```sql
Insert into tb_test values(1,'Tom'),(2,'Cat'),(3,'Jerry');
```

- 手动控制事务

```sql
start transaction;
insert into tb_test values(1,'Tom'),(2,'Cat'),(3,'Jerry');
insert into tb_test values(4,'Tom'),(5,'Cat'),(6,'Jerry');
insert into tb_test values(7,'Tom'),(8,'Cat'),(9,'Jerry');
commit;
```

>避免每次插入一条频繁开启和关闭事务

- 主键顺序插入，性能要高于乱序插入



## **大批量插入数据**

如果一次性需要插入大批量数据(比如: 几百万的记录)，使用insert语句插入性能较低，此时可以使用MySQL数据库提供的load指令进行插入。

```sql
-- 客户端连接服务端时，加上参数 -–local-infile
mysql –-local-infile -u root -p

-- 设置全局参数local_infile为1，开启从本地加载文件导入数据的开关
set global local_infile = 1;

-- 执行load指令将准备好的数据，加载到表结构中
load data local infile '/root/sql1.log' into table tb_user fields
terminated by ',' lines terminated by '\n' ;
```





# 主键优化

- 主键顺序插入不会出现`页分裂`，乱序插入会出现
- 删除数据超过一个page的一半时，会触发页合并

> 索引设计原则：
>
> - 满足业务需求的情况下，尽量降低主键的长度。
>
> - 插入数据时，尽量选择顺序插入，选择使用AUTO_INCREMENT自增主键。尽量不要使用UUID做主键或者是其他自然主键，如身份证号。
>
> - 业务操作时，避免对主键的修改





# order by优化

MySQL的排序，有两种方式：

- Using filesort : 通过表的索引或全表扫描，读取满足条件的数据行，然后在排序缓冲区sort buffer中完成排序操作，所有不是通过索引直接返回排序结果的排序都叫 FileSort 排序。

- Using index : 通过有序索引顺序扫描直接返回有序数据，这种情况即为 using index，不需要额外排序，操作效率高。



进行以下的SQL分析：

- 创建完tb_user后，执行以下sql，发现是Using filesort

```sql
explain select id,age,phone from tb_user order by age ;
```

- 再执行以下SQL，还是Using filesort

```sql
explain select id,age,phone from tb_user order by age, phone ;
```

>由于 age, phone 都没有索引，所以此时再排序时，出现Using filesort， 排序性能较低

- 创建索引

```sql
create index idx_user_age_phone on tb_user(age,phone);
```

- 创建索引后，根据age, phone进行升序排序，为Using index

```sql
explain select id,age,phone from tb_user order by age;
```

- 以下SQL也变为Using index

```sql
explain select id,age,phone from tb_user order by age , phone;
```

-  创建索引后，根据age, phone进行降序排序

```sql
explain select id,age,phone from tb_user order by age desc , phone desc ;
```

>也出现 Using index， 但是此时Extra中出现了 Backward index scan，这个代表反向扫描索引
>
>因为在MySQL中我们创建的索引，默认索引的叶子节点是从小到大排序的
>
>而此时我们查询排序时，是从大到小，所以，在扫描时，就是反向扫描

- 根据phone，age进行升序排序，phone在前，age在后

```sql
explain select id,age,phone from tb_user order by phone , age;
```

>不满足最左前缀，也出现Using filesort

- 根据age, phone进行降序一个升序，一个降序

```sql
explain select id,age,phone from tb_user order by age asc , phone desc ;
```

>创建索引时，如果未指定顺序，默认都是按照升序排序的
>
>而查询时，一个升序，一个降序，此时就会出现Using filesort

- 为了解决上述问题，可以创建一个age 升序排序，phone 倒序排序的联合索引

```sql
create index idx_user_age_phone_ad on tb_user(age asc ,phone desc);
```

- 然后再次执行如下SQL，变为Using index了

```sql
explain select id,age,phone from tb_user order by age asc , phone desc ;
```

> order by优化原则:

- 根据排序字段建立合适的索引，多字段排序时，也遵循最左前缀法则。

- 尽量使用覆盖索引。

- 多字段排序, 一个升序一个降序，此时需要注意联合索引在创建时的规则（ASC/DESC）。

- 如果不可避免的出现filesort，大数据量排序时，可以适当增大排序缓冲区大小sort_buffer_size(默认256k)。





# group by优化

- 接下来，在没有索引的情况下，执行如下SQL，查询执行计划

```sql
explain select profession , count(*) from tb_user group by profession ;
```

>发现是using temporary，用到了临时表，性能较差

- 针对于 profession ， age， status 创建一个联合索引

```sql
create index idx_user_pro_age_sta on tb_user(profession , age , status);
```

- 再执行前面相同的SQL查看执行计划，此时为using index

```sql
explain select profession , count(*) from tb_user group by profession ;
```

- 仅按照age分组

```sql
explain select profession , count(*) from tb_user group by age ;
```

>如果仅仅根据age分组，就会出现 Using temporary ；
>
>而如果是 根据profession,age两个字段同时分组，则不会出现 Using temporary。
>
>原因是因为对于分组操作，在联合索引中，也是符合最左前缀法则的





# limit优化

>在数据量比较大时，如果进行limit分页查询，在查询时，越往后，分页查询效率越低

- 因为，当在进行分页查询时，如果执行 limit 2000000,10 ，此时需要MySQL排序前2000010 记录

- 仅仅返回 2000000 - 2000010 的记录，其他记录丢弃，查询排序的代价非常大 

>优化思路：覆盖索引（不回表）+子查询（二次查出详情）

```sql
explain select * from tb_sku t , (select id from tb_sku order by id limit 2000000,10) a where t.id = a.id;
```





# count优化

| **count**用法 | **含义**                                                     |
| ------------- | ------------------------------------------------------------ |
| count(主键)   | InnoDB 引擎会遍历整张表，把每一行的 主键id 值都取出来，返回给服务层。服务层拿到主键后，直接按行进行累加(主键不可能为null) |
| count(字段)   | 没有not null 约束 : InnoDB 引擎会遍历整张表把每一行的字段值都取出来，返回给服务层，服务层判断是否为null，不为null，计数累加。有not null 约束：InnoDB 引擎会遍历整张表把每一行的字段值都取出来，返回给服务层，直接按行进行累加。 |
| count(数字)   | InnoDB 引擎遍历整张表，但不取值。服务层对于返回的每一行，放一个数字“1”进去，直接按行进行累加。 |
| count(*)      | InnoDB引擎并不会把全部字段取出来，而是专门做了优化，不取值，服务层直接按行进行累加。 |

>按照效率排序的话，count(字段) < count(主键 id) < count(1) ≈ count(*)，所以尽量使用 count(*)





# update优化

>InnoDB的行锁是针对索引加的锁，不是针对记录加的锁 ,并且该索引不能失效，否则会从行锁升级为表锁 。

- 所以update的时候，最好根据索引字段进行更新，否则行锁升级为表锁