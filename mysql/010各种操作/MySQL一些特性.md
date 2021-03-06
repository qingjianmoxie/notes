## 分区表
- 对用户来说，分布表是独立的逻辑表，但底层由多个物理子表组成
- 实现分区的代码实际上是对一层底层表的句柄对象的封装，多分区表的请求，都会通过句柄对象转化成对存储引擎的接口调用
- 分区对应用是透明的，但底层文件每个分区表都有一个使用# 分隔命名的表文件
- MySQL没有全局索引（Oracle可以定义分区索引和全局索引）
- 以下场景适合使用分区表
1. 表非常大以至于无法全部放入内存，或者只在表的最后部分有热点数据，其他均是历史数据
2. 分区表容易维护：想批量删除大量数据可以清除整个分区，可以对独立分区进行优化、检查、修复等操作
3. 分区表数据可以分布在不同的物理设备上
4. 可以使用分区表避免某些特殊的瓶颈，如innodb单索引的互斥访问，==ext3文件系统的inode竞争【这个搜索一下】==
5. 可以备份和恢复独立的分区，大数据场景下效果非常好
- 分区表的限制：
1. 一个表最多1024个分区
2. 在MySQL5.1中，分区表达式必须是整数或者返回整数值的表达式；MySQL5.5中某些场景可以直接使用列进行分区
3. 如果分区字段中有主键或者唯一索引的列，那么所有主键列和唯一索引的列都必须包含进来
4. 分区表中无法使用外键约束
#### 分区表原理
- 分区表由多个相关底层表实现，底层表由句柄对象（handler object ）表示，所以可以直接访问各个分区
- 分区表的各个索引定义都一样
- 分区的操作逻辑
1. SELECT：分区层先打开并锁住所有的底层表，优化器过滤分区，然后调用存储引擎接口访问各个分区数据
2. INSERT：分区层先打开并锁住所有的底层表，确定哪个分区接收这条记录，然后写记录到对应分区
3. DELETE：分区层先打开并锁住所有的底层表，确定数据对应的分区，然后删除对应底层表的数据
4. UPDATE：分区层先打开并锁住所有的底层表，确定更新记录所在分区，然后取出数据更新，更新后再判断数据放哪个分区，再对底层表操作，并对原数据所在的底层表做删除
- 每个操作都会先“分区层先打开并锁住所有的底层表”，但存储引擎实现自己的行级锁，则会在分区层释放对应表锁
#### 分区表类型
- 可以有多种分区类型
1. RANGE分区：基于属于一个给定连续区间的列值，把多行分配给分区。
2. LIST分区：类似于按RANGE分区，区别在于LIST分区是基于列值匹配一个离散值集合中的某个值来进行选择。
3. HASH分区：基于用户定义的表达式的返回值来进行选择的分区，该表达式使用将要插入到表中的这些行的列值进行计算。这个函数可以包含MySQL中有效的、产生非负整数值的任何表达式。
4. KEY分区：类似于按HASH分区，区别在于KEY分区只支持计算一列或多列，且MySQL服务器提供其自身的哈希函数。必须有一列或多列包含整数值。

- range分区

```
CREATE TABLE sales(
order_date DATETIME NOT NULL
-- other columns omitted
)ENGINE=InnoDB PARTITION BY RANGE (YEAR(order_date))(
PARTITION P_2010 VALUES LESS THAN(2010),
PARTITION P_2011 VALUES LESS THAN(2011),
PARTITION P_2012 VALUES LESS THAN(2012),
PARTITION P_catchall VALUES LESS THAN MAXVALUE )
```
1. partition 分区子句中可以使用各种函数，但表达式返回的要是一个确定的整数且不能是常数
2. 使用时间分区，很常见
- MySQL支持子分区，但在生产环境很少见到
- 在MySQL5.5中，还可以使用range columns类型的分区，这种类型无需将时间分区转化成整数
- 子分区可以降低索引的互斥访问竞争，因此适用于最近一年数据等被频繁访问的表或者分区
- 常见的分区类型还有：
1. 根据键值分区，减少innodb的互斥量竞争
2. 使用数学模函数分区，然后数据轮询放入不同的分区（比如对日期做mod7的运算）
3. 假如表有一个自增的主键列id，希望根据时间将最近热点数据集中存放，则时间戳必须包含在主键当中。则可以使用HASH(ID DIV 100000)将100万数据
#### 如何使用分区表

```
如果数据有10T，远大于内存，且使用传统硬盘，如何查询
首先，肯定不能每次查询都全表扫描
其次，考虑索引在空间维护上的消耗，也不希望使用索引（会有碎片产生，一个查询会产生成千上万的随机IO）
所以，只能让所有查询都只在数据表上做顺序扫描或者将数据表和索引都全部缓存在内存
```
- 数据量超大的时候，B-TREE索引就无法起作用了（除非是覆盖索引），因为会产生大量随机IO，索引维护的代价也非常高
- infobright就完全放弃了B-Tree索引，使用了更粗粒度消耗更少的方式（如大量数据上只索引对应的一小块元数据）
- 保证大数据量的可扩展性，可使用
1. 全量扫描数据，不用任何索引，但根据分区规则大致定位到数据位置（使用where条件）。
2. 索引数据，分离热点。将热点数据单独放一个分区中，然这部分数据有机会缓存在内存中，这样查询就只访问一个很小的分区表
#### 什么情况下会出问题
- null值会使得分区过滤无效（第一个分区是特殊分区，假如分区键为null或者非法值，记录会被存放到第一个分区；mysql使用where过滤，除了命中where条件的分区，还会检查第一个分区）
- 分区列和索引列不匹配。由于不能通过分区过滤，这样会导致查询时搜索全部分区的索引；在关联查询中，分区表是关联顺序中的第二个表，关联使用的索引和分区标条件不匹配也会访问分区表的所有分区
- 选择分区的成本可能会高。尤其是范围分区，查询符合条件的分区可能会代价比较高（大多数情况下，分区个数大于100）。键分区和哈希分区，没有这个问题
- 打开并锁住所有底层表的成本可能很高，尤其是对于操作很快的查询
- 维护分区的成本可能很高：新增或者删除分区速度比较快，而重组分区或者类似alter操作
- 其他限制：
1. 所有分区引擎必须相同
2. 分区函数可以使用的函数和表达式也有限制
3. 某些存储引擎不支持分区
4. 对于Myisam表，不能使用load index into cache
5. 对于Myisam表，使用分区表时需要打开更多的文件描述符，可能会出现文件描述符超过限制的问题
#### 查询优化
- 分区最大优点是优化器可以根据分区函数过滤一些分区，所以where条件很重要

```
不加where条件，扫描了全部分区
> explain select * from sales;
+----+-------------+-------+---------------------------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table | partitions                      | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+-------+---------------------------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | sales | P_2010,P_2011,P_2012,P_catchall | ALL  | NULL          | NULL | NULL    | NULL |    3 |   100.00 | NULL  |
+----+-------------+-------+---------------------------------+------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

加where条件
> explain select * from sales where order_date > "2018-08-02";
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | sales | P_catchall | ALL  | NULL          | NULL | NULL    | NULL |    2 |    50.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

但只能通过分区函数的列本身进行比较，否则不能过滤
> explain select * from sales where year(order_date) > 2018;
+----+-------------+-------+---------------------------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions                      | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+---------------------------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | sales | P_2010,P_2011,P_2012,P_catchall | ALL  | NULL          | NULL | NULL    | NULL |    3 |   100.00 | Using where |
+----+-------------+-------+---------------------------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

#### 合并表
合并表是早期的简单的分区实现，和分区表相比有一些不同的限制且缺乏优化

```
CREATE TABLE t1(a INT NOT NULL PRIMARY KEY)ENGINE=MyISAM;
creaTE TABLE t2 (a INT NOT NULL PRIMARY KEY)ENGINE =MyISAM;
INSERT INTo t1(a) VALUES(1),(2);
INSERT INTO t2(a)VALUES (1),(2);
CREATE TABLE mrg(a INT NOT NULL PRIMARY KEY) ENGINE=MERGE UNION=(t1, t2)INSERT_METHOD=LAST;

> SELECT a FROM mrg;
ERROR 1168 (HY000): Unable to open underlying table which is differently defined or of non-MyISAM type or doesn't exist

> drop table mrg;
Query OK, 0 rows affected (0.01 sec)

> CREATE TABLE mrg(a INT NOT NULL PRIMARY KEY) ENGINE=MyISAM UNION=(t1, t2)INSERT_METHOD=LAST;
Query OK, 0 rows affected (0.01 sec)

> SELECT a FROM mrg;
Empty set (0.00 sec)

```

## 视图
- MySQL5.0之后引入视图，视图本身是虚拟表，不存放任何数据
- 不能对视图创建触发器，也不能使用drop table命令删除视图
- 视图实现的方法有两种：
1. 将select语句的结果放到临时表中（有性能问题，优化器很难优化这个临时表上的查询）
2. 重写含有视图的查询，将视图的定义SQL直接包含进查询SQL中（更好的办法）
3. 使用临时表的方法称为临时表算法，重写的方法称为合并算法（MERGE）。MySQL按需采用
![合并算法和临时表算法](pic/MySQL%E4%B8%80%E4%BA%9B%E7%89%B9%E6%80%A71.png)
4. 如果使用临时表算法实现，explain中会显示为派生表（derived）
5. ==如果视图中包含group by 、distinct、聚集函数、union、子查询等，只要无法在原表记录和视图记录中建立一一映射的场景中，MySQL都会使用临时表算法实现==【实际上并非如此】

```
> create table test_view3 as select sum(c_w_id)  from customer group by c_w_id ;
Query OK, 5 rows affected (1.31 sec)
Records: 5  Duplicates: 0  Warnings: 0

> explain select * from test_view3 ;
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table      | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | test_view3 | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    5 |   100.00 | NULL  |
+----+-------------+------------+------------+------+---------------+------+---------+------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

#### 可更新视图
- 可更新视图（updatable view）是指可以通过更新视图来更新视图涉及的相关表
- 如果视图中包含了含group by 、distinct、聚集函数、union以及一些特殊情况，就不能更新
#### 视图对性能的影响
- 某些情况下，视图也能帮助提升性能
1. 重构schema的时候可以使用视图，使得在修改视图底层表结构的时候，应用代码还能继续不报错运行
2. 可以使用视图实现基于列的权限控制，却不需要真正在系统中创建列权限，因此没额外开销
#### 视图的限制
- MySQL没有物化视图
- MySQL没有视图索引
- 可以使用flexviews实现模拟物化视图和索引
## 外键
- innodb是MySQL唯一支持外键的内置存储引擎
- 外键要求每次修改数据时都要在另外一张表中执行一次查找操作
- innodb强制外键使用索引
## 在MySQL内部存储代码
#### 存储过程和函数
#### 触发器
#### 事件
#### 在存储过程中保留注释
## 游标
## 绑定变量
#### 绑定变量的优化
#### SQL接口的绑定变量
#### 绑定变量的限制
## 用户自定义函数
## 插件
## 字符集和校对规则
#### MySQL如何使用字符集
#### 选择字符集和校对规则
#### 字符集和校对规则如何影响查询
## 全文索引
#### 自然语言的全文索引
#### 布尔全文索引
#### MySQL5.1中全文索引的变化
#### 全文索引的限制和替代方案
#### 全文索引的配置和优化
## 分布式（XA）事务
#### 内部XA事务
#### 外部XA事务
## 查询缓存
#### MySQL如何判断缓存命中
#### 查询缓存如何使用内存
#### 什么情况下查询缓存能发挥作用
#### 如何配置和维护查询缓存
#### innodb和查询缓存
#### 通用查询缓存优化
#### 查询缓存的替代方案