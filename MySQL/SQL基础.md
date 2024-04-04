# SQL基础——SQL语句

### 一、DDL语句

#### 1、创建语句

- 建库语句

```mysql
# create database [ if not exists ] 数据库名 [ default charset 字符集 ] [ collate 排序规则 ] ;
create database info;
```



- 建表语句

```mysql
create table emp(
    id int comment '编号',
    workno varchar(10) comment '工号',
    name varchar(10) comment '姓名',
    gender char(1) comment '性别',
    age tinyint unsigned comment '年龄',
    idcard char(18) comment '身份证号',
    entrydate date comment '入职时间'
) comment '员工表';
```



#### 2、查询语句

- 查询所有数据库

```mysql
show databases ;
```



- 查询当前数据库

```mysql
select database() ;
```



-  查询当前数据库所有表

```mysql
 show tables;
```



- 查看看指定表结构

```mysql
desc 表名 ;
```



- 查询指定表的建表语句

```mysql
show create table 表名 ;
```



#### 3、删除语句

- 删除数据库

```mysql
drop database [ if exists ] 数据库名 ;
```



- 删除数据库中的表

```mysql
 drop table emp;
```



#### 4、char，varchar

- char定长，varchar不定长（指定的长度，char不满会填充，varchar不会）
  - 若超出，超出部分都不会取到
- char存英文一个字节，汉字两个字节，varchar都是两个字节
- char的效率略微高一点



#### 5、表操作

- 添加字段（增加属性）

```mysql
# ALTER TABLE 表名 ADD 字段名 类型 (长度) [ COMMENT 注释 ] [ 约束 ];
ALTER TABLE emp ADD nickname varchar(20) COMMENT '昵称';
```



- 修改数据类型

```mysql
# ALTER TABLE 表名 MODIFY 字段名 新数据类型 (长度)
```



-  修改字段名和字段类型

```mysql
# ALTER TABLE 表名 CHANGE 旧字段名 新字段名 类型 (长度) [ COMMENT 注释 ] [ 约束 ];
ALTER TABLE emp CHANGE nickname username varchar(30) COMMENT '昵称';
```



- 删除字段

```mysql
ALTER TABLE 表名 DROP 字段名;
```

- 可见，==drop可用于删除数据库，数据表，以及数据表中的字段（行）==



- 修改表名

```mysql
ALTER TABLE 表名 RENAME TO 新表名;
```



- 删除表

```mysql
DROP TABLE [ IF EXISTS ] 表名;
```



- 删除指定表, 并重新创建表

```mysql
TRUNCATE TABLE 表名;	#表中的数据也会清空，执行完后就是空表（结构还是一样的，数据都没了）
```



### 二、DML语句（数据增删改）

#### 1、增（INSERT）

> 注意事项：

-  插入数据时，指定的字段顺序需要与值的顺序是一一对应的。

- 字符串和日期型数据应该包含在引号中。

-  插入的数据大小，应该在字段的规定范围内。



##### 方式一:

- 写多少字段，对应的value需要有几个

```mysql
# INSERT INTO 表名 (字段名1, 字段名2, ...) VALUES (值1, 值2, ...);
insert into employee(id,workno,name,gender,age,idcard,entrydate)
values(1,'1','Itcast','男',10,'123456789012345678','2000-01-01');
```



##### 方式二：

- 默认全部字段，value需要全部填上

```mysql
# INSERT INTO 表名 VALUES (值1, 值2, ...);
insert into employee values(2,'2','张无忌','男',18,'123456789012345670','2005-01-
01');
```



##### 方式三：

- 批量插入数据（两种方法），不同的数据间用逗号

```mysql
#法一：
INSERT INTO 表名 (字段名1, 字段名2, ...) VALUES (值1, 值2, ...), (值1, 值2, ...), (值1, 值2, ...) ;

#法二：
INSERT INTO 表名 VALUES (值1, 值2, ...), (值1, 值2, ...), (值1, 值2, ...) ;
```





#### 2、改

- 修改数据的具体语法为：

```mysql
UPDATE 表名 SET 字段名1 = 值1 , 字段名2 = 值2 , .... [ WHERE 条件 ] ;
```

>注意事项: 
>
>- 修改语句的条件可以有，也可以没有，如果没有条件，则会修改整张表的所有数据。
>- 不同字段间用，隔开



##### 示例：

- 修改id为1的数据，将name修改为itheima

```mysql
update employee set name = 'itheima' where id = 1;
```



-  修改id为1的数据, 将name修改为小昭, gender修改为女

```mysql
update employee set name = '小昭' , gender = '女' where id = 1;
```



- 将所有的员工入职日期修改为 2008-01-01

```mysql
update employee set entrydate = '2008-01-01';
```



#### 3、删

- 删除数据的具体语法为：

```mysql
DELETE FROM 表名 [ WHERE 条件 ] ;
```



>注意事项:
>
>- DELETE是删除满足指定条件的行，即将这一个数据从表中删除（一个数据很多列）
>
>-  DELETE 语句的条件可以有，也可以没有，如果没有条件，则会删除整张表的所有数据。
>- DELETE 语句不能删除某一个字段的值(可以使用UPDATE，将该字段值置为NULL即 可)。



##### 示例：

- 删除gender为女的员工

```mysql
delete from employee where gender = '女';
```



- 删除所有员工

```mysql
delete from employee;
```





### 三、DQL语句（查）

> 在一个正常的业务系统中，查询操作的频次是要远高于增删改的。而且在查询的过程中，可能 还会涉及到条件、排序、分页等操作。



#### 1、基本语法

```mysql
SELECT
		字段列表
FROM
		表名列表
WHERE
		条件列表
GROUP BY
		分组字段列表
HAVING
		分组后条件列表
ORDER BY
		排序字段列表
LIMIT
		分页参数

```



讲解这部分内容的时候，会将上面的完整语法进行拆分，分为以下几个部分：

>- 基本查询（不带任何条件）
>-  条件查询（WHERE）
>-  聚合函数（count、max、min、avg、sum） 
>- 分组查询（group by）
>-  排序查询（order by） 
>- 分页查询（limit）



其中，表结构如下：

```mysql
mysql> select * from emp;
+------+--------+---------+--------+------+--------------------+-------------+------------+
| id   | workno | name    | gender | age  | idcard             | workaddress | entrydate  |
+------+--------+---------+--------+------+--------------------+-------------+------------+
|    1 | 00001  | 柳岩666 | 女     |   20 | 123456789012345678 | 北京        | 2000-01-01 |
|    2 | 00002  | 张无忌  | 男     |   18 | 123456789012345670 | 北京        | 2005-09-01 |
|    3 | 00003  | 韦一笑  | 男     |   38 | 123456789712345670 | 上海        | 2005-08-01 |
|    4 | 00004  | 赵敏    | 女     |   18 | 123456757123845670 | 北京        | 2009-12-01 |
|    5 | 00005  | 小昭    | 女     |   16 | 123456769012345678 | 上海        | 2007-07-01 |
|    6 | 00006  | 杨逍    | 男     |   28 | 12345678931234567X | 北京        | 2006-01-01 |
|    7 | 00007  | 范瑶    | 男     |   40 | 123456789212345670 | 北京        | 2005-05-01 |
|    8 | 00008  | 黛绮丝  | 女     |   38 | 123456157123645670 | 天津        | 2015-05-01 |
|    9 | 00009  | 范凉凉  | 女     |   45 | 123156789012345678 | 北京        | 2010-04-01 |
|   10 | 00010  | 陈友谅  | 男     |   53 | 123456789012345670 | 上海        | 2011-01-01 |
|   11 | 00011  | 张士诚  | 男     |   55 | 123567897123465670 | 江苏        | 2015-05-01 |
|   12 | 00012  | 常遇春  | 男     |   32 | 123446757152345670 | 北京        | 2004-02-01 |
|   13 | 00013  | 张三丰  | 男     |   88 | 123656789012345678 | 江苏        | 2020-11-01 |
|   14 | 00014  | 灭绝    | 女     |   65 | 123456719012345670 | 西安        | 2019-05-01 |
|   15 | 00015  | 胡青牛  | 男     |   70 | 12345674971234567X | 西安        | 2018-04-01 |
|   16 | 00016  | 周芷若  | 女     |   18 | NULL               | 北京        | 2012-06-01 |
+------+--------+---------+--------+------+--------------------+-------------+------------+
```



#### 2、基本查询（不带任何条件）

##### 1）查询多个字段

```mysql
SELECT 字段1, 字段2, 字段3 ... FROM 表名 ;
```

```mysql
SELECT * FROM 表名 ;
```

>==不同字段间用，隔开==
>
>\* 号代表查询所有字段，在实际开发中尽量少用（影响效率、不直观，即看不到具体属性字段）。



##### 2）字段设置别名

```mysql
SELECT 字段1 [ AS 别名1 ] , 字段2 [ AS 别名2 ] ... FROM 表名;
```

```mysql
SELECT 字段1 [ 别名1 ] , 字段2 [ 别名2 ] ... FROM 表名;
```



##### 3）去重

```mysql
SELECT DISTINCT 字段列表 FROM 表名;
```



##### 4）案例

-  查询指定字段 name, workno, age并返回

```mysql
 select name,workno,age from emp;
```



- 查询返回所有字段

```mysql
select * from emp;
```



-  查询所有员工的工作地址,起别名

```mysql
 select workaddress as '工作地址' from emp;
 -- as可以省略
 select workaddress '工作地址' from emp;
```



- 查询公司员工的上班地址有哪些(不要重复)

```mysql
 select distinct workaddress from emp;
```



#### 3、条件查询

##### 1）语法

```mysql
SELECT 字段列表 FROM 表名 WHERE 条件列表 ;
```



##### 2）条件

>比较运算符：\> ，>=，<，<=，=，！=（<>），BETWEEN...AND...，IN(...)，LIKE（模糊匹配），is null
>
>逻辑运算符：AND（&&），OR（||），NOT（!）



##### 3）案例

- 查询年龄小于等于 20 的员工信息

```mysql
select * from emp where age <= 20;
```



- 查询没有身份证号的员工信息

```mysql
select * from emp where idcard is null;
```



-  查询年龄在15岁(包含) 到 20岁(包含)之间的员工信息

```mysql
select * from emp where age between 15 and 20;
select * from emp where age >= 15 and age <= 20;
```



-  查询性别为 女 且年龄小于 25岁的员工信息

```mysql
select * from emp where gender = '女' and age < 25;
```



-  查询年龄等于18 或 20 或 40 的员工信息

```mysql
select * from emp where age in(18,20,40);
select * from emp where age = 18 or age = 20 or age =40;
```



-  查询姓名为两个字的员工信息 _ %

```mysql
 select * from emp where name like '__';
```



-  查询身份证号最后一位是X的员工信息

```mysql
select * from emp where idcard like '%X';
```



#### 4、聚合函数

##### 1）介绍

将一列数据作为一个整体，进行纵向计算 。（==对列操作==）



##### 2）常见的聚合函数

>count ： 统计数量
>
>max ：最大值
>
>min ： 最小值
>
>avg ：平均值
>
>sum ：求和



##### 3）语法

```mysql
SELECT 聚合函数(字段列表) FROM 表名 ;
```

>注意：
>
>- 聚合函数的结果作为select查询的值，需要放在select后面
>-  NULL值是不参与所有聚合函数运算的。



##### 4）案例

- 统计该企业员工数量

```mysql
select count(*) from emp; -- 统计的是总记录数，结果为16
select count(idcard) from emp; -- 统计的是idcard字段不为null的记录数，结果为15，有一条null不参与运算
```

>对于count聚合函数，统计符合条件的总记录数



-  统计该企业员工的平均年龄

```mysql
select avg(age) from emp;
```



- 统计该企业员工的最大年龄

```mysql
select max(age) from emp;
```



- 统计西安地区员工的年龄之和

```mysql
select sum(age) from emp where workaddress = '西安';
```



#### 5、==分组查询==

##### 1）语法

```mysql
SELECT 字段列表 FROM 表名 [ WHERE 条件 ] GROUP BY 分组字段名 [ HAVING 分组后过滤条件 ];
```



##### 2）==where与having区别==

- 执行时机不同：where是分组之前进行过滤，不满足where条件，不参与分组；而having是分组 之后对结果进行过滤。
- 判断条件不同：where不能对聚合函数进行判断，而having可以。

>注意事项:
>
>- 执行顺序: where > 聚合函数 > having
>
>-  支持多字段分组, 具体语法为 : group by columnA,columnB（逗号分隔）



##### 3）案例

- 根据性别分组 , 统计男性员工和女性员工的数量

```mysql
 select count(*) from emp groug by gender;
 
 +----------+
| count(*) |
+----------+
|        7 |
|        9 |
+----------+

select gender,count(*) from emp group by gender;

+--------+----------+
| gender | count(*) |
+--------+----------+
| 女     |        7 |
| 男     |        9 |
+--------+----------+
```



-  根据性别分组 , 统计男性员工和女性员工的平均年龄

```mysql
 select gender,avg(age) from emp group by gender;
```



-  ==查询年龄小于45的员工 , 并根据工作地址分组 , 获取员工数量大于等于3的工作地址==

```mysql
# 首先查询年龄小于45的员工
select * from emp where age < 45;
+------+--------+---------+--------+------+--------------------+-------------+------------+
| id   | workno | name    | gender | age  | idcard             | workaddress | entrydate  |
+------+--------+---------+--------+------+--------------------+-------------+------------+
|    1 | 00001  | 柳岩666 | 女     |   20 | 123456789012345678 | 北京        | 2000-01-01 |
|    2 | 00002  | 张无忌  | 男     |   18 | 123456789012345670 | 北京        | 2005-09-01 |
|    3 | 00003  | 韦一笑  | 男     |   38 | 123456789712345670 | 上海        | 2005-08-01 |
|    4 | 00004  | 赵敏    | 女     |   18 | 123456757123845670 | 北京        | 2009-12-01 |
|    5 | 00005  | 小昭    | 女     |   16 | 123456769012345678 | 上海        | 2007-07-01 |
|    6 | 00006  | 杨逍    | 男     |   28 | 12345678931234567X | 北京        | 2006-01-01 |
|    7 | 00007  | 范瑶    | 男     |   40 | 123456789212345670 | 北京        | 2005-05-01 |
|    8 | 00008  | 黛绮丝  | 女     |   38 | 123456157123645670 | 天津        | 2015-05-01 |
|   12 | 00012  | 常遇春  | 男     |   32 | 123446757152345670 | 北京        | 2004-02-01 |
|   16 | 00016  | 周芷若  | 女     |   18 | NULL               | 北京        | 2012-06-01 |
+------+--------+---------+--------+------+--------------------+-------------+------------+

# 再按照工作地址分组
select * from emp where age < 45 group by workaddress;
+------+--------+---------+--------+------+--------------------+-------------+------------+
| id   | workno | name    | gender | age  | idcard             | workaddress | entrydate  |
+------+--------+---------+--------+------+--------------------+-------------+------------+
|    1 | 00001  | 柳岩666 | 女     |   20 | 123456789012345678 | 北京        | 2000-01-01 |
|    3 | 00003  | 韦一笑  | 男     |   38 | 123456789712345670 | 上海        | 2005-08-01 |
|    8 | 00008  | 黛绮丝  | 女     |   38 | 123456157123645670 | 天津        | 2015-05-01 |
+------+--------+---------+--------+------+--------------------+-------------+------------+

# 发现结果不对，因为对上述第一部查出来的年龄小于45的员工，对工作地址分组后分为3组（有3个工作地址）
# select * 对于分组后意义不明，就把符合条件的各个组内第一个结果返回了
# 所以有group by的sql语句对应的select一般查询的为聚合函数（对分组后的数据条目求聚合函数）

#正确的语句应该如下：
 select count(*) from emp where age < 45 group by workaddress;
 +----------+
| count(*) |
+----------+
|        7 |
|        2 |
|        1 |
+----------+

#加上地址信息
select workaddress,count(*) from emp where age < 45 group by workaddress;
+-------------+----------+
| workaddress | count(*) |
+-------------+----------+
| 北京        |        7 |
| 上海        |        2 |
| 天津        |        1 |
+-------------+----------+
# 可见，统计的结果为符合where条件的数据按工作地址分组后各组数据的总数目
# 借此加深理解，先执行where，group by是对where后筛选的结果进行分组的

#最后，获取员工数量大于等于3的工作地址
 select workaddress,count(*) from emp where age < 45 group by workaddress having count(*) >= 3;
 +-------------+----------+
| workaddress | count(*) |
+-------------+----------+
| 北京        |        7 |
+-------------+----------+

# having字句中可以带聚合函数，但是where字句中不行
```



- 统计各个工作地址上班的男性及女性员工的数量（==多字段分组==）

```mysql
 select workaddress,gender,count(*) from emp group by gender,workaddress;
```



#### 6、排序查询

##### 1）语法

```mysql
SELECT 字段列表 FROM 表名 ORDER BY 字段1 排序方式1 , 字段2 排序方式2 ;
```



##### 2）排序方式

- ASC：升序（默认）
- DESC：降序

>注意事项：
>
>- 如果是升序, 可以不指定排序方式ASC ;
>-  如果是多字段排序，当第一个字段值相同时，才会根据第二个字段进行排序 ;



##### 3）案例

- 根据年龄对公司的员工进行升序排序

```mysql
 select * from emp order by age asc;
select * from emp order by age;
```



- 根据入职时间, 对员工进行降序排序

```mysql
 select * from emp order by entrydate desc;
```



- 根据年龄对公司的员工进行升序排序 , 年龄相同 , 再按照入职时间进行降序排序

```mysql
select * from emp order by age asc,entrydate desc;
```



#### 7、分页查询

##### 1）语法

```mysql
SELECT 字段列表 FROM 表名 LIMIT 起始索引, 查询记录数 ;
```

>注意事项:
>
>- 起始索引从0开始，起始索引 = （查询页码 - 1）* 每页显示记录数。
>
>- 如果查询的是第一页数据，起始索引可以省略，直接简写为 limit 10。（这里10表示每页显示记录数）



##### 2）案例

- 查询第1页员工数据, 每页展示10条记录

```mysql
 select * from emp limit 0,10;
 select * from emp limit 10;
```



- 查询第2页员工数据, 每页展示10条记录 --------> (页码-1)*页展示记录数

```mysql
 select * from emp limit 10,10;
```



#### 8、练习

-  查询年龄为20,21,22,23岁的女员工信息。

```mysql
select * from emp where gender = '女' and age in(20,21,22,23);
```



-  查询性别为 男 ，并且年龄在 20-40 岁(含)以内的姓名为三个字的员工。

```mysql
select * from emp where gender = '男' and (age between 20 and 40) and name like '___';
```



-  统计员工表中, 年龄小于60岁的 , 男性员工和女性员工的人数。

```mysql
select gender, count(*) from emp where age < 60 group by gender;
```



- 查询所有年龄小于等于35岁员工的姓名和年龄，并对查询结果按年龄升序排序，如果年龄相同按 入职时间降序排序。

```mysql
 select name,age from emp where age <= 35 order by age asc,entrydate desc;
```



- 查询性别为男，且年龄在20-40 岁(含)以内的前5个员工信息，对查询的结果按年龄升序排序， 年龄相同按入职时间升序排序。

```mysql
select * from emp where gender = '男' and (age between 20 and 40) order by age asc,entrydate asc limit 5;
```



#### 9、DQL执行顺序

![image-20220807133640291](C:\Users\mcw\AppData\Roaming\Typora\typora-user-images\image-20220807133640291.png)





### 四、DCL语句（数据控制语句）

不关键，就是创建用户、修改密码以及权限控制等语句

