# SQL基础——约束

==有关SQL函数的直接看基础篇的PDF，就不手写了==



### 一、概述

- 概念：约束是作用于表上字段的规则，用于限制存储在表中的数据

- 目的：保证数据库中数据的正确、有效性和完整性

- 分类：

| 约束     | 描述                                               | 关键字      |
| -------- | -------------------------------------------------- | ----------- |
| 非空约束 | 限制该字段的数据不能为null                         | NOT NULL    |
| 唯一约束 | 保证该字段的所有数据都是唯一、不重复的             | UNIQUE      |
| 主键约束 | 主键是一行数据的唯一标识，要求非空且唯一           | PRIMARY KEY |
| 默认约束 | 保存数据时，如果未指定该字段的值，则采用默认值     | DEFAULT     |
| 检查约束 | 保证字段满足某一个条件（check检查的条件）          | CHECK       |
| 外键约束 | 用于两张表的数据建立连接，保证数据的一致性和完整性 | FOREIGN KEY |

>注意：约束是作用于表中字段上的，可以在创建表/修改表的时候添加约束。



### 二、约束演示

- 案例需求： 根据需求，完成表结构的创建。需求如下：

| 字段名 | 字段含义     | 字段类型    | 约束条件                  | 约束关键字                 |
| :----- | ------------ | ----------- | ------------------------- | -------------------------- |
| id     | ID唯一标识符 | int         | 主键，且自动增长          | PRIMARY KEY AUTO_INCREMENT |
| name   | 姓名         | varchar(10) | 不为空，且唯一            | NOT NULL , UNIQUE          |
| age    | 年龄         | int         | 大于0，并且小于等于120    | CHECK                      |
| status | 状态         | char(1)     | 如果没有指定该值，默认为1 | DEFAULT                    |
| gender | 性别         | char(1)     | 无                        |                            |

- 对应的建表语句如下：

```mysql
CREATE TABLE tb_user(
    id int AUTO_INCREMENT PRIMARY KEY COMMENT 'ID唯一标识',
    name varchar(10) NOT NULL UNIQUE COMMENT '姓名' ,
    age int check (age > 0 && age <= 120) COMMENT '年龄' ,
    status char(1) default '1' COMMENT '状态',
    gender char(1) COMMENT '性别'
);
```



### 三、==外键约束==

#### 1、添加外键

##### 1）建表时

```mysql
CREATE TABLE 表名(
    字段名 数据类型,
    ...
    [CONSTRAINT] [外键名称] FOREIGN KEY (外键字段名) REFERENCES 主表 (主表列名)
);
```

##### 2）修改表

```mysql
ALTER TABLE 表名 ADD CONSTRAINT 外键名称 FOREIGN KEY (外键字段名) REFERENCES 主表 (主表列名) ;
```



##### 3）案例

- 为emp表的dept_id字段添加外键约束,关联dept表的主键id

```mysql
alter table emp add constraint fk_emp_dept_id foreign key (dept_id) references dept(id)
```

>注意事项：
>
>- 添加了外键约束后，不能删除或更新父表记录（该记录有外键时），因为存在外键约束。



### 2、删除外键

```mysql
ALTER TABLE 表名 DROP FOREIGN KEY 外键名称;
```

```mysql
alter table emp drop foreign key fk_emp_dept_id;
```



### 3、删除/更新行为

##### 1）类别

| 行为        | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| NO ACTION   | 当在父表中删除/更新对应记录时，首先检查该记录是否有对应外键，如果有则不允许删除/更新。 (与 RESTRICT 一致) 默认行为 |
| RESTRICT    | 当在父表中删除/更新对应记录时，首先检查该记录是否有对应外键，如果有则不允许删除/更新。 (与 NO ACTION 一致) 默认行为 |
| CASCADE     | 当在父表中删除/更新对应记录时，首先检查该记录是否有对应外键，如果有，则也删除/更新外键在子表中的记录。 |
| SET NULL    | 当在父表中删除对应记录时，首先检查该记录是否有对应外键，如果有则设置子表中该外键值为null（这就要求该外键允许取null）。 |
| SET DEFAULT | 父表有变更时，子表将外键列设置成一个默认的值 (Innodb不支持)  |



##### 2）语法

- 添加外键时，设定删除/更新行为的具体语法为：

```mysql
ALTER TABLE 表名 ADD CONSTRAINT 外键名称 FOREIGN KEY (外键字段) REFERENCES 主表名 (主表字段名) ON UPDATE CASCADE ON DELETE CASCADE;
```



##### 3）案例

-  CASCADE

```mysql
alter table emp add constraint fk_emp_dept_id foreign key (dept_id) references dept(id) on update cascade on delete cascade;
```



- SET NULL

```mysql
alter table emp add constraint fk_emp_dept_id foreign key (dept_id) references dept(id) on update set null on delete set null;
```



### 四、多表查询

#### 1、多表关系

各个表结构之间也存在着各种联系，基本上分为三种：

- 一对多(多对一)
- 多对多
- 一对一



##### 1）一对多

- 案例：部门和员工的关系
- 关系：一个部门对应多个员工，一个员工对应一个部门
- 实现：在多的一方建立外键，指向一的一方的主键

![image-20220807162144362](C:\Users\mcw\AppData\Roaming\Typora\typora-user-images\image-20220807162144362.png)



##### 2）多对多

- 案例：学生与课程的关系
- 关系：一个学生可以选修多个课程，一门课程也供多个学生选修
- 实现：建立第三张中间表，中间表至少包含两个外键，分别关联两方主键

![image-20220807162320516](C:\Users\mcw\AppData\Roaming\Typora\typora-user-images\image-20220807162320516.png)

- 中间表的建表SQL如下：

```mysql
create table student_course(
    id int auto_increment comment '主键' primary key,
    studentid int not null comment '学生ID',
    courseid int not null comment '课程ID',
    constraint fk_courseid foreign key (courseid) references course (id),
    constraint fk_studentid foreign key (studentid) references student (id)
)comment '学生课程中间表';

```



##### 3）一对一

- 案例：用户与用户详情的关系
- 关系：一对一关系，多用于单表拆分，将一张表的基础字段放在一张表中，其他详情字段放在另 一张表中，以提升操作效率
- 实现：在任意一方加入外键，关联另外一方的主键，并且设置外键为唯一的(UNIQUE)

![image-20220807162521590](C:\Users\mcw\AppData\Roaming\Typora\typora-user-images\image-20220807162521590.png)

>注意事项：
>
>- 本来是一张表，将其拆分为上图的两张表，以提升效率



#### 2、多表查询概述

##### 1）基本语法

查询单表数据时，SQL语句如下：

```mysql
select * from emp;
```

若是多表数据，只需要使用逗号分隔开多个表即可，如

```,ysql
select * from emp, dept;
```

>此时，数据量极大：
>
>- 员工表emp所有的记录 (17) 与 部门表dept所有记录(6) 进行了组合，即笛卡尔积



- 如何消除无效的笛卡尔积（==加上连接查询的条件==）

```mysql
select * from emp, dept where emp.dept_id = dept.id;
```



##### 2）分类

- 连接查询
  - 内连接：查询A、B交集部分数据
  - 外连接
    - 左外连接：查询左表所有数据，以及两张表交集部分数据
    - 右外连接：查询右表所有数据，以及两张表交集部分数据
  - 自连接：当前表与自身的连接查询，自连接必须使用表别名
- 子查询（表示多个SELECT嵌套查询）



#### 3、数据准备

- 两张表的数据如下所示，其中emp表的dept_id作为外键连接dept表的主键

```mysql
mysql> select * from emp;
+----+--------+------+--------------+--------+------------+-----------+---------+
| id | name   | age  | job          | salary | entrydate  | managerid | dept_id |
+----+--------+------+--------------+--------+------------+-----------+---------+
|  1 | 金庸   |   66 | 总裁         |  20000 | 2000-01-01 |      NULL |       5 |
|  2 | 张无忌 |   20 | 项目经理     |  12500 | 2005-12-05 |         1 |       1 |
|  3 | 杨逍   |   33 | 开发         |   8400 | 2000-11-03 |         2 |       1 |
|  4 | 韦一笑 |   48 | 开发         |  11000 | 2002-02-05 |         2 |       1 |
|  5 | 常遇春 |   43 | 开发         |  10500 | 2004-09-07 |         3 |       1 |
|  6 | 小昭   |   19 | 程序员鼓励师 |   6600 | 2004-10-12 |         2 |       1 |
|  7 | 灭绝   |   60 | 财务总监     |   8500 | 2002-09-12 |         1 |       3 |
|  8 | 周芷若 |   19 | 会计         |  48000 | 2006-06-02 |         7 |       3 |
|  9 | 丁敏君 |   23 | 出纳         |   5250 | 2009-05-13 |         7 |       3 |
| 10 | 赵敏   |   20 | 市场部总监   |  12500 | 2004-10-12 |         1 |       2 |
| 11 | 鹿杖客 |   56 | 职员         |   3750 | 2006-10-03 |        10 |       2 |
| 12 | 鹤笔翁 |   19 | 职员         |   3750 | 2007-05-09 |        10 |       2 |
| 13 | 方东白 |   19 | 职员         |   5500 | 2009-02-12 |        10 |       2 |
| 14 | 张三丰 |   88 | 销售总监     |  14000 | 2004-10-12 |         1 |       4 |
| 15 | 俞莲舟 |   38 | 销售         |   4600 | 2004-10-12 |        14 |       4 |
| 16 | 宋远桥 |   40 | 销售         |   4600 | 2004-10-12 |        14 |       4 |
| 17 | 陈友谅 |   42 | NULL         |   2000 | 2011-10-12 |         1 |    NULL |
+----+--------+------+--------------+--------+------------+-----------+---------+

mysql> select * from dept;
+----+--------+
| id | name   |
+----+--------+
|  1 | 研发部 |
|  2 | 市场部 |
|  3 | 财务部 |
|  4 | 销售部 |
|  5 | 总经办 |
|  6 | 人事部 |
+----+--------+
```

>注意事项：
>
>- 陈友谅目前没有部门，若是内连接查询，该记录没法查询到（不属于交集部分）
>- 但是可以通过外连接查询到，该条数据属于emp表内的数据



#### 4、内连接

- 查询两张表交集部分的数据

##### 1）隐式内连接

```mysql
SELECT 字段列表 FROM 表1 , 表2 WHERE 条件 ... ;
```



##### 2）显式内连接

```mysql
SELECT 字段列表 FROM 表1 [ INNER ] JOIN 表2 ON 连接条件 ... ;
```



##### 3）案例

- 查询每一个员工的姓名 , 及关联的部门的名称 (隐式内连接实现)

```mysql
select emp.name,dept.name from emp,dept where emp.dept_id = dept.id;

-- 为每一张表起别名,简化SQL编写(先执行from，在那里起别名前面select能用上)
select e.name,d.name from emp e , dept d where e.dept_id = d.id;
```



-  查询每一个员工的姓名 , 及关联的部门的名称 (显式内连接实现) 

```mysql
select e.name,d.name from emp e inner join dept d on e.dept_id = d.id; 

-- inner可以省略
select e.name,d.name from emp e join dept d on e.dept_id = d.id; 
```



#### 5、外连接

外连接分为两种：

- 左外连接：查询左表的所有数据，当然也包含交集部分的数据。

```mysql
SELECT 字段列表 FROM 表1 LEFT [ OUTER ] JOIN 表2 ON 条件 ... ;
```

- 右外连接：查询右表的所有数据，当然也包含交集部分的数据。

```,ysql
SELECT 字段列表 FROM 表1 RIGHT [ OUTER ] JOIN 表2 ON 条件 ... ;
```

>注意事项：
>
>- 本质上左外连接和右外连接一样，实际开发中一般用左外连接，将右外换顺序即为左外连接



##### 1）案例

- 查询emp表的所有数据, 和对应的部门信息（因为陈友谅，不能内连接）

```mysql
select e.*,d.name from emp e left outer join dept d on e.dept_id = d.id;

-- outer可以省略
select e.*, d.name from emp e left join dept d on e.dept_id = d.id;
```



-  查询dept表的所有数据, 和对应的员工信息(右外连接)

```mysql
select d.*,e.* from emp e right outer join dept d on e.dept_id = d.id; 
```

- 注意，from表示以什么为基准，相当于emp在左边，去右连接查询右表的所有数据

>注意事项：
>
>- 左外连接和右外连接是可以相互替换的，只需要调整在连接查询时SQL中，表结构的先后顺 序就可以了。而我们在日常开发使用时，更偏向于左外连接。
>- from的就想象成左边的表，再根据具体查询的数据选择怎么连接（想象韦恩图）



#### 6、自连接

自连接查询，就是自己连接自己，也就是把一张表连接查询多次。

##### 1）语法

```mysql
SELECT 字段列表 FROM 表A 别名A JOIN 表A 别名B ON 条件 ... ;
```

>注意事项：
>
>- 对于自连接查询，可以是内连接查询，也可以是外连接查询。



##### 2）案例

- 查询员工及其所属领导的名字，分析如下：
  - 每条数据有一个managerid字段，表示其领导id
  - 领导也是职员，该id在emp表中也表现为一条数据
  - 需要连接的信息其实是在同一张表内的，但仅利用单表查询无法实现需求



- 首先利用隐式内连接查询，如下：

```mysql
select a.name, b.name from emp a,emp b where a.managerid = b.id;
```



- 观察表结构，金庸没有领导，内连接查交集无法查到该记录，采用左外连接，如下：

```mysql
select a.name,b.name from emp a left join emp b on a.managerid = b.id;

-- 为了更直观，可以给字段取别名
select a.name '员工', b.name '领导' from emp a left join emp b on a.managerid = b.id;
```

>注意事项：
>
>- 在自连接查询中，必须要为表起别名，要不然我们不清楚所指定的条件、返回的字段，到底 是哪一张表的字段。





#### 7、联合查询

对于union查询，就是把多次查询的结果合并起来，形成一个新的查询结果集。

##### 1）语法：

```mysql
SELECT 字段列表 FROM 表A ...
UNION [ ALL ]
SELECT 字段列表 FROM 表B ....;
```

>注意事项：
>
>- 对于联合查询的多张表的列数必须保持一致，字段类型也需要保持一致。
>
>- union all 会将全部的数据直接合并在一起，union 会对合并之后的数据去重。



##### 2）案例

- 将薪资低于 5000 的员工 , 和 年龄大于 50 岁的员工全部查询出来
  - 当前对于这个需求，我们可以直接使用多条件查询，使用逻辑运算符 or 连接即可
  - 也可以通过union/union all来联合查询

```mysql
mysql> select * from emp where salary < 5000
    -> union all
    -> select * from emp where age > 50;
```

- 发现有2个鹿杖客，因为他两个条件都满足（union all都选了出来）

```mysql
mysql> select * from emp where salary < 5000
    -> union
    -> select * from emp where age > 50;
```



#### 8、子查询

##### 1）概述

概念：SQL语句中嵌套SELECT语句，称为嵌套查询，又称子查询。

```mysql
SELECT * FROM t1 WHERE column1 = ( SELECT column1 FROM t2 );
```



##### 2）分类

根据子查询结果不同，分为：

- 标量子查询（子查询结果为单个值）
- 列子查询(子查询结果为一列)
- 行子查询(子查询结果为一行)
- 表子查询(子查询结果为多行多列)

根据子查询位置，分为：

- WHERE之后
- FROM之后
- SELECT之后



##### 3）标量子查询

- 子查询返回的结果是单个值（数字、字符串、日期等），最简单的形式

- 常用的操作符：= <> > >= < <= 



> 案例

- 查询 "销售部" 的所有员工信息，可见子查询结果为标量

```mysql
select * from emp where dept_id = (select id from dept where name = '销售部');

# 其中，子查询结果为：
mysql> select id from dept where name = '销售部';
+----+
| id |
+----+
|  4 |
+----+
```



- 查询在 "方东白" 入职之后的员工信息

```mysql
select * from emp where entrydate > (select entrydate from emp where name = '方东白');
```



##### 4）列子查询

- 子查询返回的结果是一列（可以是多行），这种子查询称为列子查询。
- 常用的操作符：IN 、NOT IN 、 ANY 、SOME 、 ALL

| 操作符 | 描述                                   |
| ------ | -------------------------------------- |
| IN     | 在指定的集合范围之内，多选一           |
| NOT IN | 不在指定的集合范围之内                 |
| ANY    | 子查询返回列表中，有任意一个满足即可   |
| SOME   | 与ANY等同，使用SOME的地方都可以使用ANY |
| ALL    | 子查询返回列表的所有值都必须满足       |



>案例：

 查询 "销售部"和"市场部"的所有员工信息：

- ①查询 "销售部" 和 "市场部" 的部门ID，可见子查询结果为列

```mysql
select id from dept where name = '销售部' or name = '市场部';

+----+
| id |
+----+
|  2 |
|  4 |
+----+
```

- ②根据部门ID, 查询员工信息

```mysql
select * from emp where dept_id in (select id from dept where name = '销售部' or name = '市场部');
```



查询比财务部所有人工资都高的员工信息：

- ①查询所有财务部人员工资

```mysql
select id from dept where name = '财务部';

select salary from emp where dept_id = (select id from dept where name = '财务部');
```

- ②比财务部所有人工资都高的员工信息

```mysql
select * from emp where salary > all (select salary from emp where dept_id = (select id from dept where name = '财务部'));
```



查询比研发部其中任意一人工资高的员工信息：

- ①查询研发部所有人工资

```mysql
select salary from emp where dept_id = (select id from dept where name = '研发部');
```

- ②比研发部其中任意一人工资高的员工信息

```mysql
select * from emp where salary > any ( select salary from emp where dept_id = (select id from dept where name = '研发部') );
```



##### 5）行子查询

- 子查询返回的结果是一行（可以是多列），这种子查询称为行子查询。
- 常用的操作符：= 、<> 、IN 、NOT IN

>案例

查询与"张无忌"的薪资及直属领导相同的员工信息：

- ①查询"张无忌"的薪资及直属领导

```mysql
mysql> select salary,managerid from emp where name = '张无忌';

# 可见，子查询结果为行
+--------+-----------+
| salary | managerid |
+--------+-----------+
|  12500 |         1 |
+--------+-----------+
```

-  ②查询与 "张无忌" 的薪资及直属领导相同的员工信息

```mysql
select * from emp where (salary,managerid) = (select salary,managerid from emp where name = '张无忌');
```



##### 6）表子查询

- 子查询返回的结果是多行多列，这种子查询称为表子查询。
- 常用的操作符：IN

>案例

查询与 "鹿杖客" , "宋远桥" 的职位和薪资相同的员工信息：

- ①查询"鹿杖客","宋远桥"的职位和薪资

```mysql
mysql> select job,salary from emp where name = '鹿杖客' or name = '宋远桥';

# 可见，子查询的结果为表结构
+------+--------+
| job  | salary |
+------+--------+
| 职员 |   3750 |
| 销售 |   4600 |
+------+--------+
```

- ②查询与"鹿杖客","宋远桥"的职位和薪资相同的员工信息

```mysql
select * from emp where (job,salary) in (select job,salary from emp where name = '鹿杖客' or name = '宋远桥');
```



查询入职日期是 "2006-01-01" 之后的员工信息 , 及其部门信息：

- ①入职日期是 "2006-01-01" 之后的员工信息

```mysql
select * from emp where entrydate > '2006-01-01';
```

- ②查询这部分员工, 对应的部门信息（因为陈友谅没有部门，也要查到，必须外连接）

```mysql
select e.*,d.* from (select * from emp where entrydate > '2006-01-01') e left join dept d on e.dept_id = d.id;
```



### 五、多表查询练习

三个表的结构如下：

```mysql
mysql> select * from emp;
+----+--------+------+--------------+--------+------------+-----------+---------+
| id | name   | age  | job          | salary | entrydate  | managerid | dept_id |
+----+--------+------+--------------+--------+------------+-----------+---------+
|  1 | 金庸   |   66 | 总裁         |  20000 | 2000-01-01 |      NULL |       5 |
|  2 | 张无忌 |   20 | 项目经理     |  12500 | 2005-12-05 |         1 |       1 |
|  3 | 杨逍   |   33 | 开发         |   8400 | 2000-11-03 |         2 |       1 |
|  4 | 韦一笑 |   48 | 开发         |  11000 | 2002-02-05 |         2 |       1 |
|  5 | 常遇春 |   43 | 开发         |  10500 | 2004-09-07 |         3 |       1 |
|  6 | 小昭   |   19 | 程序员鼓励师 |   6600 | 2004-10-12 |         2 |       1 |
|  7 | 灭绝   |   60 | 财务总监     |   8500 | 2002-09-12 |         1 |       3 |
|  8 | 周芷若 |   19 | 会计         |  48000 | 2006-06-02 |         7 |       3 |
|  9 | 丁敏君 |   23 | 出纳         |   5250 | 2009-05-13 |         7 |       3 |
| 10 | 赵敏   |   20 | 市场部总监   |  12500 | 2004-10-12 |         1 |       2 |
| 11 | 鹿杖客 |   56 | 职员         |   3750 | 2006-10-03 |        10 |       2 |
| 12 | 鹤笔翁 |   19 | 职员         |   3750 | 2007-05-09 |        10 |       2 |
| 13 | 方东白 |   19 | 职员         |   5500 | 2009-02-12 |        10 |       2 |
| 14 | 张三丰 |   88 | 销售总监     |  14000 | 2004-10-12 |         1 |       4 |
| 15 | 俞莲舟 |   38 | 销售         |   4600 | 2004-10-12 |        14 |       4 |
| 16 | 宋远桥 |   40 | 销售         |   4600 | 2004-10-12 |        14 |       4 |
| 17 | 陈友谅 |   42 | NULL         |   2000 | 2011-10-12 |         1 |    NULL |
+----+--------+------+--------------+--------+------------+-----------+---------+


mysql> select * from dept;
+----+--------+
| id | name   |
+----+--------+
|  1 | 研发部 |
|  2 | 市场部 |
|  3 | 财务部 |
|  4 | 销售部 |
|  5 | 总经办 |
|  6 | 人事部 |
+----+--------+


mysql> select * from salgrade;
+-------+-------+-------+
| grade | losal | hisal |
+-------+-------+-------+
|     1 |     0 |  3000 |
|     2 |  3001 |  5000 |
|     3 |  5001 |  8000 |
|     4 |  8001 | 10000 |
|     5 | 10001 | 15000 |
|     6 | 15001 | 20000 |
|     7 | 20001 | 25000 |
|     8 | 25001 | 30000 |
+-------+-------+-------+
```



1、查询员工的姓名、年龄、职位、部门信息 （隐式内连接）

- 表: emp , dept
- 连接条件: emp.dept_id = dept.id

```mysql
select e.name,e.age,e.job,d.name from emp e,dept d where e.dept_id = d.id;
```



2、查询年龄小于30岁的员工的姓名、年龄、职位、部门信息（显式内连接）

- 表: emp , dept 

- 连接条件: emp.dept_id = dept.id

```mysql
select e.name , e.age , e.job , d.name from emp e inner join dept d on e.dept_id = d.id where e.age < 30;
```



3、 查询拥有员工的部门ID、部门名称

-  表：emp , dept
- 连接条件: emp.dept_id = dept.id

```mysql
select distinct d.id, d.name from emp e,dept d where e.dept_id = d.id;
```



4、查询所有年龄大于40岁的员工, 及其归属的部门名称; 如果员工没有分配部门, 也需要展示出来(外连接)

-  表：emp , dept
- 连接条件: emp.dept_id = dept.id

```mysql
select e.*,d.name from emp e left join dept d on e.dept_id = d.id where e.age > 40;
```



5、 查询所有员工的工资等级

- 表: emp , salgrade
- 连接条件：emp.salary between salgrade.losal and salgrade.hisal

```mysql
select e.*,s.grade from emp e,salgrade s where e.salary between s.losal and s.hisal;
```



6、 查询 "研发部" 所有员工的信息及工资等级

-  表：emp , salgrade , dept
- 连接条件：emp.salary between salgrade.losal and salgrade.hisal，emp.dept_id = dept.id
- 查询条件：dept.name = '研发部'

```mysql
select e.*,s.grade from emp e,salgrade s,dept d where e.dept_id = d.id and (e.salary between s.losal and hisal) and d.name = '研发部';
```



7、 查询 "研发部" 员工的平均工资

- 表：emp , dept
- 连接条件 : emp.dept_id = dept.id

```mysql
select avg(e.salary) from emp e,dept d where e.dept_id = d.id and d.name = '研发部';
```



8、 查询工资比 "灭绝" 高的员工信息

-  查询 "灭绝" 的薪资

```mysql
select e.salary from emp e where e.name = '灭绝';
```

- 查询比她工资高的员工数据

```mysql
select * from emp e where e.salary > (select e.salary from emp e where e.name = '灭绝');
```



9、 查询比平均薪资高的员工信息

- 查询员工的平均薪资

```mysql
select avg(salary) from emp;
```

- 查询比平均薪资高的员工信息

```mysql
select * from emp where salary > (select avg(salary) from emp);
```



10、查询低于本部门平均工资的员工信息

-  查询指定部门平均薪资

```mysql
select avg(e1.salary) from emp e1 where e1.dept_id = 1;
select avg(e1.salary) from emp e1 where e1.dept_id = 2;
```

-  查询低于本部门平均工资的员工信息

```mysql
select * from emp e2 where e2.salary < (select avg(salary) from emp e1 where e2.dept_id = e1.dept_id);
```



==11、查询所有的部门信息, 并统计部门的员工人数==

```mysql
select d.id,d.name,(select count(*) from emp e where e.dept_id = d.id) from dept d;
```



12、 查询所有学生的选课情况, 展示出学生名称, 学号, 课程名称

- 表: student , course , student_course
- 连接条件：student.id = student_course.studentid，course.id = student_course.courseid

```mysql
select s.name,s.no,c.name from student s,course c,student_course sc where s.id = sc.studentid and c.id = sc.courseid;
```

