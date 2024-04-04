# Redis三大特殊数据类型

### geospatial地理位置

- 一共6个命令
  - GEOADD
  - GEODIST
  - GEOHASH
  - GEOPOS
  - GEORADIUS
  - GEORADIUSBYMEMBER



#### GEOADD

```bash
127.0.0.1:6379> geoadd china:city 116.40 39.90 beijing
(integer) 1
127.0.0.1:6379> geoadd china:city 121.47 31.23 shanghai
(integer) 1
127.0.0.1:6379> geoadd china:city 106.50 29.53 chongqing
(integer) 1
127.0.0.1:6379> geoadd china:city 114.05 22.52 shenzhen
(integer) 1
127.0.0.1:6379> geoadd china:city 120.16 30.24 hangzhou
(integer) 1
127.0.0.1:6379> geoadd china:city 108.96 34.26 xian
(integer) 1
```



#### GEOPOS

```bash
127.0.0.1:6379> GEOPOS china:city beijing
1) 1) "116.39999896287918091"
   2) "39.90000009167092543"
127.0.0.1:6379> GEOPOS china:city beijing chongqing
1) 1) "116.39999896287918091"
   2) "39.90000009167092543"
2) 1) "106.49999767541885376"
   2) "29.52999957900659211"
```



#### GEODIST

- 计算距离

```bash
127.0.0.1:6379> geodist china:city beijing shanghai
"1067378.7564"
127.0.0.1:6379> geodist china:city beijing shanghai km
"1067.3788"
```



#### GEORADIUS

- 获取半径内的元素，应用：我附近的人

```bash
127.0.0.1:6379> GEORADIUS china:city 110 30 1000 km
1) "chongqing"
2) "xian"
3) "shenzhen"
4) "hangzhou"
```



- 显示到中间距离的位置

```bash
127.0.0.1:6379> GEORADIUS china:city 110 30 500 km withdist
1) 1) "chongqing"
   2) "341.9374"
2) 1) "xian"
   2) "483.8340"
```



- 显示他人的定位信息

```bash
127.0.0.1:6379> GEORADIUS china:city 110 30 1000 km withcoord
1) 1) "chongqing"
   2) 1) "106.49999767541885376"
      2) "29.52999957900659211"
2) 1) "xian"
   2) 1) "108.96000176668167114"
      2) "34.25999964418929977"
3) 1) "shenzhen"
   2) 1) "114.04999762773513794"
      2) "22.5200000879503861"
4) 1) "hangzhou"
   2) 1) "120.1600000262260437"
      2) "30.2400003229490224"

```



- 查询指定数量的元素

```bash
127.0.0.1:6379> GEORADIUS china:city 110 30 1000 km withdist withcoord count 2
1) 1) "chongqing"
   2) "341.9374"
   3) 1) "106.49999767541885376"
      2) "29.52999957900659211"
2) 1) "xian"
   2) "483.8340"
   3) 1) "108.96000176668167114"
      2) "34.25999964418929977"
```



#### GEORADIUSBYMEMBER

- 和上述命令差不多，就是把中心点变成指定位置
- 找出位于指定元素周围的其他元素

```bash
127.0.0.1:6379> GEORADIUSBYMEMBER china:city beijing 1000 km
1) "beijing"
2) "xian"
```



#### GEOHASH

- 将二维的经纬度转换成一维的字符串（Hashcode）

```bash
127.0.0.1:6379> GEOHASH china:city beijing
1) "wx4fbxxfke0"
```



#### GEO底层的实现就是Zset，可通过Zset的API操作GEO

```bash
127.0.0.1:6379> ZRANGE china:city 0 -1
1) "chongqing"
2) "xian"
3) "shenzhen"
4) "hangzhou"
5) "shanghai"
6) "beijing"
127.0.0.1:6379> zrem china:city beijing
(integer) 1
127.0.0.1:6379> ZRANGE china:city 0 -1
1) "chongqing"
2) "xian"
3) "shenzhen"
4) "hangzhou"
5) "shanghai"
```



### Hyperloglog

- 什么是基数
- A{1，3，4，7，8，7}
- B{1，3，5，7，8}
- 基数（不重复的元素）= 5，可以接受误差
- 命令以PF开头



![image](https://user-images.githubusercontent.com/106053649/181045426-22b69d4d-56e4-44c4-a865-af43c57a05f4.png)



- 基本的命令

```bash
127.0.0.1:6379> pfadd mykey a b c d e f g h i j
(integer) 1
127.0.0.1:6379> PFCOUNT mykey
(integer) 10
127.0.0.1:6379> pfadd mykey2 i j z x c v b n m
(integer) 1
127.0.0.1:6379> PFCOUNT mykey2
(integer) 9
127.0.0.1:6379> PFMERGE mykey3 mykey mykey2
OK
127.0.0.1:6379> PFCOUNT mykey3
(integer) 15
```

- 如果允许容错，此类基数统计一定可以用Hyperloglog
- 如果不允许出错，就使用set或者自己的数据类型即可



### Bitmap

- 位存储（两个状态的），操作二进制位来进行记录
- 场景：统计用户信息，活跃，不活跃！登录、未登录！打卡！



#### 例如打卡

- 使用setbit命令，sign表示打卡记录，0表示周一以此类推，最后的0表示不打卡，1表示打卡

```bash
127.0.0.1:6379> setbit sign 0 1
(integer) 0
127.0.0.1:6379> setbit sign 1 1
(integer) 0
127.0.0.1:6379> setbit sign 2 1
(integer) 0
127.0.0.1:6379> setbit sign 3 1
(integer) 0
127.0.0.1:6379> setbit sign 4 1
(integer) 0
127.0.0.1:6379> setbit sign 5 0
(integer) 0
127.0.0.1:6379> setbit sign 6 0
(integer) 0
```



- getbit操作

```bash
127.0.0.1:6379> getbit sign 3
(integer) 1
```



- bitcount统计1的记录个数

```bash\
127.0.0.1:6379> bitcount sign
(integer) 5
```

