# Redis五大数据类型

### Redis-Key  

- 通过EXISTS key可以查看是否存在对应key的字段

```bash
EXISTS name	//查看是否有key为name的数据
```

- 可以通过move key 1将对应key的数据移动到数据库1

```bash
move name 1	//将key为name的数据从当前数据库移动到数据库1
```

- 设置某个数据的过期时间（缓存热点数据、cookie等）
  - 可通过ttl key查看还剩余多少时间

```bash
EXPIRE name 10	//即10秒后key为name的数据将会过期
```

- 通过type key可以查看当前key的类型

```bash
type name
```



### String（字符串）

#### 基本操作

- APPEND key可以往key对应的value中添加字符串，若不存在则相当于set

```bash
127.0.0.1:6379> set key1 v1
OK
127.0.0.1:6379> get key1
"v1"
127.0.0.1:6379> EXISTS key1
(integer) 1
127.0.0.1:6379> APPEND key1 "hello"
(integer) 7
127.0.0.1:6379> get key1
"v1hello"
```

- STRLEN key可以查看key对应的value的字符串长度

```bash
STRLEN key1		//结果得到7
```



#### 增减操作

- 可以通过incr命令让某个key对应的value增加
  - 虽然类型都是字符串，但是其value要看起来像数字（不然会报错）
  - incrby key 10可以一次加10

```bash
127.0.0.1:6379> set view 0
OK
127.0.0.1:6379> INCR view
(integer) 1
127.0.0.1:6379> type view
string
```

- 对应地，可以通过decr key让其减少
  - decrby key 5就是一次减少5

```bash
DECRBY view 5
```



#### 获取字串

- 通过GETRANGE获取key对应value的子串

```bash
GETRANGE key start end	//end若为-1表示全部
```

- 通过SETRANGE将偏移量后面替换（替换字串多长就替换多少）

```bash
set key abcdefg
get key				//得到abcdefg
SETRANGE key 1 xx	//将下标1开始替换为xx
get key				//得到axxdefg
```



#### setex和setnx（很重要）

- setex（set with expire）表示设置过期时间,即同时具备set及expire功能

- setnx（set if no exist）不存在则设置==（分布式锁中常使用）==

```bash
127.0.0.1:6379> setex name 10 mcw
OK
127.0.0.1:6379> ttl name
(integer) 8
127.0.0.1:6379> setnx mykey redis
(integer) 1
127.0.0.1:6379> keys *
1) "mykey"
127.0.0.1:6379> ttl name
(integer) -2
127.0.0.1:6379> setnx mykey mongoDB
(integer) 0								//0表示设置失败
127.0.0.1:6379> get mykey
"redis"
127.0.0.1:6379>
```



#### 批量设置（查看）值

- mset批量设置

```bash
127.0.0.1:6379> mset k1 v1 k2 v2 k3 v3
OK
127.0.0.1:6379> keys *
1) "k1"
2) "k3"
3) "k2"
127.0.0.1:6379>
```

- mget批量查看

```bash
127.0.0.1:6379> mget k1 k2 k3
1) "v1"
2) "v2"
3) "v3"
127.0.0.1:6379>
```



- 同理对应也有msetnx
  - 考虑在上述情况下，有了k1，k2，k3
  - 如果设置msetnx k1 v1 k4 v4，按照正常理解k1存在不创建，k4不存在创建
  - 但实际上不会穿件k4，因为msetnx是原子性操作（要么都做、要么都不做）

```bash
127.0.0.1:6379> msetnx k1 v1 k4 v4
(integer) 0
```



#### 对象操作

- 常规的操作（不推荐）

```bash
set user:1 {name:zhangsan, age:3}	#设置一个user:1对象，值为json字符串来保存一个对象
```

- 通过对key进行巧妙设计（有点类似key的递归）如下：

```bash
127.0.0.1:6379> mset user:1:name zhangsan user:1:age 2
OK
127.0.0.1:6379> mget user:1:name user:1:age
1) "zhangsan"
2) "2"
```

- 本质是将user:1:name作为key进行set和get
  - 本身user可以有很多user，这样设计很符合实际
    - user有点像结构体数组，每个uesr都很很多属性
  - 首先通过1拿到user1
  - 再通过user1：name拿到其对应的属性



#### getset方法

- getset表示取得key对应的value并设置新的值，若是没有则会创建

```bash
127.0.0.1:6379> getset name mcw
(nil)
127.0.0.1:6379> get name
"mcw"
127.0.0.1:6379> getset name mmmcw
"mcw"
127.0.0.1:6379> get name
"mmmcw"
```



### List

- Redis中将List实现成栈、队列、阻塞队列

- List的所有命令都是以l开头的



#### 基本操作

- 插入操作： LPUSH key value [value ...]，是往头部插入
  - 操作的本质是将List这种类型作为key

```bash
127.0.0.1:6379> LPUSH list one
(integer) 1
127.0.0.1:6379> LPUSH list two
(integer) 2
127.0.0.1:6379> LPUSH list three
(integer) 3
127.0.0.1:6379> keys *				//可见，当前只有一个key为L
1) "list"
```



- 范围查询：LRANGE（Redis不区分大小写）

```bash
127.0.0.1:6379> Lrange list 0 -1
1) "three"
2) "two"
3) "one"
```



- 从另一侧插入，尾部插入（Rpush）

```bash
127.0.0.1:6379> Lrange list 0 -1
1) "three"
2) "two"
3) "one"
4) "right"
//可以看到一开始插一个前面的往后移，最先插入的one在最后，rpush直接从尾部开始插入
```



- 移除操作POP

```BASH
127.0.0.1:6379> lpop list	//对比之前的结果，最左边即第一个元素three被pop了
"three"
127.0.0.1:6379> Lrange list 0 -1
1) "two"
2) "one"
3) "right"
127.0.0.1:6379> rpop list	//移除最后一个元素
"right"
```



- 通过下标获取list中的某一个值（Lindex）

```bash
127.0.0.1:6379> LINDEX list 1
"one"
127.0.0.1:6379> LINDEX list 0
"two"
```



- 获取List的长度（Llen）

```bash
127.0.0.1:6379> Llen list
(integer) 2
```



- 移除指定的值（Lrem）
  - 具体命令为：lrem key count value
  - 表示从名为key的list中移除对应值为value的count个数据

```bash
127.0.0.1:6379> Lpush list one
(integer) 1
127.0.0.1:6379> Lpush list one
(integer) 2
127.0.0.1:6379> Lpush list two
(integer) 3
127.0.0.1:6379> LRANGE list 0 -1
1) "two"
2) "one"
3) "one"
127.0.0.1:6379> lrem list 2 one
(integer) 2
127.0.0.1:6379> LRANGE list 0 -1
1) "two"
```



- Ltrim修剪：LTRIM key start stop，留下原List中下标为start到stop的元素（下标从0开始）

```bash
127.0.0.1:6379> LRANGE list 0 -1
1) "three"
2) "one"
3) "two"
127.0.0.1:6379> LTRIM list 1 2
OK
127.0.0.1:6379> LRANGE list 0 -1
1) "one"
2) "two"
```



- rpoplpush（移除列表的最后一个元素，并将它移动到新的列表中）

```bash
127.0.0.1:6379> LRANGE list 0 -1
1) "three"
2) "one"
3) "two"
127.0.0.1:6379> RPOPLPUSH list newlist
"two"
127.0.0.1:6379> LRANGE list 0 -1
1) "three"
2) "one"
127.0.0.1:6379> LRANGE newlist 0 -1
1) "two"
```



- lset：设定list对应下标的值（必须存在对应下标的元素）
  - LSET key index value

```bash
127.0.0.1:6379> LRANGE list 0 -1
1) "two"
2) "one"
127.0.0.1:6379> lset list 0 five
OK
127.0.0.1:6379> LRANGE list 0 -1
1) "five"
2) "one"
```



- Linsert：将某一个具体的value插入到列表中某个元素的前面或后面
  - LINSERT key BEFORE|AFTER pivot value

```bash
127.0.0.1:6379> LRANGE list 0 -1
1) "five"
2) "one"
127.0.0.1:6379> LINSERT list after five six
(integer) 3
127.0.0.1:6379> LRANGE list 0 -1
1) "five"
2) "six"
3) "one"
```



#### List小结

- 实际表现是一个链表，前后都可插入值
- 如果key不存在，创建新的链表
- 如果key存在，新增内容
- 如果移除了所有值，空链表，也代表不存在
- 在两边插入或者改动，效率最高。更改中间元素，相对来说效率低一些
- 可以实现消息排队
  - 消息队列（Lpush， Rpop）
  - 栈（Lpush， Lpop）



### Set（无序不重复集合）

- Set的命令都是S开头的



#### 添加

- sadd往set里添加元素
  - 实际命令为：SADD key member [member ...]

```bash
127.0.0.1:6379> sadd myset one
(integer) 1
127.0.0.1:6379> sadd myset two
(integer) 1
```



#### 查看

- smember：查看set里的元素

```bash
127.0.0.1:6379> SMEMBERS myset
1) "one"
2) "two"
```



#### 查看某个元素是否存在

- sismember

```bash
127.0.0.1:6379> SISMEMBER myset one
(integer) 1
```



#### 获取元素的个数

- scard ： 查看set集合中的元素个数

```bash
127.0.0.1:6379> scard myset
(integer) 2
```



#### 移除元素

- srem ：移除某个元素

```bash
127.0.0.1:6379> SMEMBERS myset
1) "one"
2) "two"
127.0.0.1:6379> srem myset one
(integer) 1
127.0.0.1:6379> SMEMBERS myset
1) "two"
```



#### 随机取元素

- SRANDMEMBER 
  - 命令为SRANDMEMBER key [count]
  - 不指定就是随机一个，指定count会随机生成指定个数的元素

```bash
127.0.0.1:6379> SMEMBERS myset
1) "five"
2) "two"
3) "four"
4) "three"
5) "one"
127.0.0.1:6379> SRANDMEMBER myset
"three"
127.0.0.1:6379> SRANDMEMBER myset 3
1) "three"
2) "five"
3) "four"
```



#### 随机删除key

- Spop随机移除元素，因为set内部无序

```bash
127.0.0.1:6379> SMEMBERS myset
1) "three"
2) "two"
3) "five"
4) "four"
127.0.0.1:6379> spop myset
"four"
127.0.0.1:6379> SMEMBERS myset
1) "three"
2) "two"
3) "five"
```



#### 将一个指定的key移到另外的set中

- Smove，命令格式为：SMOVE source destination member

```bash
127.0.0.1:6379> SMEMBERS myset
1) "two"
2) "one"
127.0.0.1:6379> SMEMBERS myset2
1) "three"
127.0.0.1:6379> SMOVE myset myset2 one
(integer) 1
127.0.0.1:6379> SMEMBERS myset
1) "two"
127.0.0.1:6379> SMEMBERS myset2
1) "one"
2) "three"
```



#### 求并集（共同关注）

- 数字集合类：

  - 差集

    - Sdiff

    - ```bash
      127.0.0.1:6379> SMEMBERS key1
      1) "b"
      2) "a"
      3) "c"
      127.0.0.1:6379> SMEMBERS key2
      1) "e"
      2) "d"
      3) "c"
      127.0.0.1:6379> SDIFF key1 key2
      1) "a"
      2) "b"
      ```

  - 交集

    - Sinter

    - ```bash
      127.0.0.1:6379> sinter key1 key2
      1) "c"
      ```

  - 并集

    - Sunion

    - ```bash
      127.0.0.1:6379> SUNION key1 key2
      1) "c"
      2) "a"
      3) "b"
      4) "e"
      5) "d"
      ```



### Hash（哈希）

- 理解为Map集合，key ->  (key - value)，本质和String类型没有区别，还是key-value
- 即通过key找到的value本身就是一个key-value对

- 所以Hash的命令以h开头



#### get及set

- set命令：HSET key field value
- 将field 与 value存成键值对的形式，通过key及field查到value

```bash
127.0.0.1:6379> HSET myhash k1 v1
(integer) 1
127.0.0.1:6379> HGET myhash k1
"v1"
```



#### hmset和hmget

- 批量set和批量get

```bash
127.0.0.1:6379> hmset myhash k1 v1 k2 v2
OK
127.0.0.1:6379> hmget myhash k1 k2
1) "v1"
2) "v2"
127.0.0.1:6379> HGETALL myhash
1) "k1"
2) "v1"
3) "k2"
4) "v2"
```



#### 删除

- hdel，删除了filed其对应的value也没了

```bash
127.0.0.1:6379> HDEL myhash k1
(integer) 1
127.0.0.1:6379> HGETALL myhash
1) "k2"
2) "v2"
```



#### 获取长度

- Hlen：获取哈希表的字段数量

```bash
127.0.0.1:6379> HLEN myhash
(integer) 1
```



#### 判断是否存在

- Hexists

```bash
127.0.0.1:6379> HEXISTS myhash k2
(integer) 1
```



#### 只获得key和只获得value

```bash
127.0.0.1:6379> HKEYS myhash
1) "k2"
127.0.0.1:6379> HVALS myhash
1) "v2"
```



#### 增加和减少

- incr和decr（incrby表示指定增量，不仅为加1）

```bash
127.0.0.1:6379> hmset myhash k1 1 k2 2
OK
127.0.0.1:6379> HINCRBY myhash k1 2
(integer) 3
```



#### Hsetnx

- 如果不存在则创建

```bash
127.0.0.1:6379> hsetnx myhash k3 v3
(integer) 1
127.0.0.1:6379> hsetnx myhash k3 v3
(integer) 0
```



#### hash应用

- hash可以用来存用户信息等经常变动的信息
- hash更适合对象的存储
- String更适合字符串的存储



### Zset（有序集合）

- 在set的基础上，增加了一个值（用来排序的标志）
  - 如set中为sadd myset  v1
  - 在zset中为zadd myset 1 v1

- 所有的命令以Z开头



#### 添加

```bash
127.0.0.1:6379> zadd myset 1 one
(integer) 1
127.0.0.1:6379> zadd myset 2 two
(integer) 1
127.0.0.1:6379> zadd myset 3 three
(integer) 1
127.0.0.1:6379> ZRANGE myset 0 -1
1) "one"
2) "two"
3) "three"
```



#### 排序

- ZRANGEBYSCORE（默认的ZRANGE会默认升序排列）

```BASH
127.0.0.1:6379> zadd salary 2500 xiaohong
(integer) 1
127.0.0.1:6379> zadd salary 1000 zhangsan
(integer) 1
127.0.0.1:6379> zadd salary 500 kuangshen
(integer) 1
127.0.0.1:6379> ZRANGEBYSCORE salary -inf +inf
1) "kuangshen"
2) "zhangsan"
3) "xiaohong"
127.0.0.1:6379> ZRANGEBYSCORE salary -inf +inf withscores
1) "kuangshen"
2) "500"
3) "zhangsan"
4) "1000"
5) "xiaohong"
6) "2500"
127.0.0.1:6379> ZRANGEBYSCORE salary -inf 1000 withscores
1) "kuangshen"
2) "500"
3) "zhangsan"
4) "1000"
127.0.0.1:6379> ZREVRANGE salary 0 -1			//降序排序
1) "zhangsan"
2) "kuangshen"
```



#### 移除元素

- zrem

```bash
127.0.0.1:6379> ZRANGE salary 0 -1
1) "kuangshen"
2) "zhangsan"
3) "xiaohong"
127.0.0.1:6379> zrem salary xiaohong
(integer) 1
127.0.0.1:6379> ZRANGE salary 0 -1
1) "kuangshen"
2) "zhangsan"
```



#### 查看元素个数

- Zcard获取有序集合中的个数

```bash
127.0.0.1:6379> zcard salary
(integer) 2
```



#### Zcount统计区间内的元素有几个

```bash
127.0.0.1:6379> ZRANGE myset 0 -1
1) "hello"
2) "world"
3) "kuangshen"
127.0.0.1:6379> ZCOUNT myset 1 2
(integer) 2
```



#### Zset应用

- 带权重的消息
- 工资表、趁机表
- 排行榜应用（Top N）

