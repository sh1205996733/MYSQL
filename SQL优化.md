# MYSQL优化记录

## IN很多时查询效率低

情景：有索引，单表数据几万条而已，

```sql
SELECT distinct user_id FROM `cash_flows`  WHERE (tenant_id = 21 and user_id >0)
```

![image-20210728175214129](SQL优化.assets\image-20210728175214129.png)

```sql
SELECT * FROM `users`  WHERE `users`.`deleted_at` IS NULL AND id in(?,?,?....?)//几万个 
mysql中，in语句中参数个数是不限制的。不过对整段sql语句的长度有了限制（max_allowed_packet）。默认是4M
IN()列表 中值的数量仅受该max_allowed_packet值的限制 
```

![image-20210728175632731](SQL优化.assets\image-20210728175632731.png)

索引失效，效率很低

References：

https://dev.mysql.com/doc/refman/5.7/en/comparison-operators.html#function_in

https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html#sysvar_max_allowed_packet

### 优化思路一（推荐）

数据库in的参数是有限制的，最好看下官方文档对in的限制个数，使用exists代替in

```sql
SELECT * FROM `users`  WHERE `users`.`deleted_at` IS NULL AND ((exists (SELECT 1 FROM `cash_flows`  WHERE (tenant_id = 21 and user_id = users.id)))) LIMIT 15
```

![image-20210728175752474](SQL优化.assets\image-20210728175752474.png)

参考链接：

[MySQL的in查询效率太低的解决办法之一与其它优化示例](https://blog.csdn.net/pengyufight/article/details/77523404)

### 优化思路二（推荐）

使用左连接

```sql
SELECT * FROM `users`  
LEFT JOIN cash_flows on user_id = users.id
WHERE `users`.`deleted_at` IS NULL LIMIT 15
```

![image-20210728180709578](SQL优化.assets\image-20210728180709578.png)

### 优化思路三

如果传进来的是一个参数列表的话 ，那么可以通过遍历的方式，循环来查,利用Limit来查，效率一般

```go
var totle []uint
var offset = 10000
databases.DbDefault.Model(cashflow.CashFlow{}).Where("tenant_id = ? and user_id > 0", i.TenantId).Pluck("distinct user_id", &totle)
count := len(totle)
index := 0
db1 := databases.DbDefault.Model(cashflow.CashFlow{})
for count > 0 {
   var id []uint
   subQuery := db1.Where("tenant_id = ? and user_id > 0", i.TenantId).Select("id").Order("id asc").Limit(1).Offset(index).SubQuery()
   db1.Where("tenant_id = ? and user_id > 0", i.TenantId).Where("id >= ?", subQuery).
      Limit(math.Min(float64(count), float64(offset))).Pluck("distinct user_id", &id)
   ids = append(ids, id...)
   index += offset
   count -= offset
   if count <= 0 {
      break
   }
}
```

## MYSQL使用时间字段索引

### MySQL5.7

MySQL5.7版本中不支持函数索引，因此 遇到函数索引的时候需要进行修改，否则即使查询的字段上有索引，执行时也无法使用索引而进行全表扫描，数据量大的表查询时间会比较长。具体案例如下

之前给test_table表的create_time加上索引

```sql
SELECT * FROM test_table WHERE DATE_FORMAT(create_time,"%Y-%m-%d") >= '2019-12-30'
```

因为使用函数时，索引会失效，用下面这种方式就可以使用索引（或者使用虚拟列的方式）

```sql
SELECT * FROM test_table WHERE create_time >= str_to_date('2019-12-30', '%Y-%m-%d')
```

### MySQL8.0

MySQL8.0的索引特性增加了函数索引。其实MySQL5.7中推出了虚拟列的功能，而MySQL8.0的函数索引也是依据虚拟列来实现的。将上述的案例在MySQL8.0中实现情况如下文所述。

```sql
alter table tb_function add key idx_create_time((date(create_time))); --   注意里面字段的括号
```

```sql
select * from  tb_function  where   date(create_time)='2020-07-01';
```

```sql
mysql> explain select  *  from  tb_function  where   date(create_time)='2020-07-01';
+----+-------------+-------------+------------+------+-----------------+-----------------+---------+-------+------+----------+-------+
| id | select_type | table       | partitions | type | possible_keys   | key             | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------------+------------+------+-----------------+-----------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | tb_function | NULL       | ref  | idx_create_time | idx_create_time | 4       | const |    3 |   100.00 | NULL  |
+----+-------------+-------------+------------+------+-----------------+-----------------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

可见，在MySQL8.0 创建对应的函数索引后，不改变SQL写法的前提下，查询的列上进行对应的函数计算后也可以走索引。

## MYSQL时间范围查询索引失效

### 问题描述

 ```sql
查询语句SQL1：
explain
 SELECT create_time
 FROM `vehicle_revision_redelivered`
 where expect_delivered_time >= '2019-10-15 00:00:00'  and expect_delivered_time<= '2019-10-30 23:59:59' ;
查询语句SQL2：
explain
 SELECT create_time
 FROM `vehicle_revision_redelivered`
 where expect_delivered_time >= '2019-11-21 00:00:00'  and expect_delivered_time<= '2020-01-30 23:59:59' ;
 ```

测试数据量：1151136 条数据,同样的sql 不同的只是查询范围不同 第一走的全表扫描，第二个走的索引

分析了一下，总数据量一共是1151136 条 ，10月15至10月30号数据量是290644条 占总数量的25.2% ，扫描行数1004564，11月21号至2020年1月30号数据量是106967，约占总数量的9.3% 扫描行数401360行,  数据主要集中在10月22至11月30号 ，这段时间的数据一共1091903条

```sql
查询语句SQL3：
explain
SELECT create_time
FROM vehicle_revision_redelivered
where expect_delivered_time >= '2019-11-20 00:00:00' and expect_delivered_time<= '2020-01-30 23:59:59' ;
```

11月20至1月30号的数据一共142514 占总数据量的12.3%，扫描行数1004564(不走索引)

```sql
查询语句SQL4：
explain
SELECT create_time
FROM vehicle_revision_redelivered
where expect_delivered_time >= '2019-11-20 00:00:00' and expect_delivered_time<= '2020-01-23 23:59:59' ;
```

11月20号至1月23号的数据一共135789条 约占总数据量的 11.8%，扫描行数426220行(走索引)

### 总结

当使用MySql 非主键索引进行查询时 如果扫描数据量接近全表数据量时，mysql会进行全表扫描不会使用索引（主键索引除外），这也是为什么不建议在区分度低的字段上建索引，也会导致全表扫描。
 rows不能直接理解为扫描行数， 表示MySQL根据表统计信息及索引选用情况，估算的找到所需的记录所需要读取的行数也就是mysql认为必须要逐行去检查和判断的记录的条数，实际是mysql根据估算的所需读取的行数决定是全表扫描还是使用索引

参考链接：

[MYSQL时间范围查询索引失效](https://www.jianshu.com/p/29f3ae7ae336)

## MYSQL日期范围查询优化

### 优化前 SQL 说明：

1. 表 A left join 表 B,Join 条件若干，最终做 count 返回条数。
2. where 条件中 create_time 总是扫描近一个月数据。单次扫描 50 万左右
3. 表 A 中整体数量 140 万条，表 B 总体数据量 140 万条。
4. 其它若干 where 条件
5. explain 返回结果表 A 扫描记录数 50 万左右，primary 索引，create_time 索引，表 B 关联 1Row，使用到了表 A 的主键关联，以及自己表中的外键索引
6. 两张表均为主键自增。

### 优化思路

1. 缩小扫描行数
2. 经过思考，最终选择了，将 create_time 扫描缩小至 1 天（大概 4000 行），也就是过去一个月开始的那天，然后求最小主键 ID（min(id) as start_id ）,再使用该 ID 作为范围起始 ID,使用表 A 的主键 id >= 该起始 ID。这样始终使时间范围扫描的是一天的数据量，这个量最大也就是万级别，不会到 10 万级以上。
3. 主键扫描，优于二级索引。

### 优化结果

1. 经测试，单表查询，这么做提升并不明显。
2. 关联查询中，多条件复合使用，这么做从单次查询 1S+降低至 100ms，有将近 10 倍的性能提升。

### 总结

1. 本次优化虽增加了一个子查询，但是整体扫描行数缩减明显， 性能提升明显，这里仅仅是提供一种减少扫描行数的优化思路。其中 SQL 细节，各个业务都不同，不一一列出。

## MYSQL慢查询优化之多范围查询优化

### 慢查询分析

```sql
SELECT COUNT(*)
FROM tb_user 
WHERE age BETWEEN 18  AND 25 
AND register_time BETWEEN 20190101 AND 20191231 
```

表中总共存在1000万数据量，在上述查询涉及的age和register_time上均已经创建了索引，使用explain分析慢查询，发现mysql使用了如下查询策略

```tex
+----+-------------+--------------+------------+-------+-----------------------------------------------------+------------------------+---------+------+---------+----------+-----------------------------------------------+
| id | select_type | table        | partitions | type  | possible_keys                                       | key                    | key_len | ref  | rows    | filtered | Extra                                         |
+----+-------------+--------------+------------+-------+-----------------------------------------------------+------------------------+---------+------+---------+----------+-----------------------------------------------+
|  1 | SIMPLE      | tb_user      | NULL       | range | age_idx,register_time_idx                           | age_idx                | 4       | NULL | 2245240 |   100.00 | Using index condition; Using where; Using MRR |
+----+-------------+--------------+------------+-------+-----------------------------------------------------+------------------------+---------+------+---------+----------+-----------------------------------------------+
```

因为mysql在执行查询的时候只会使用一个索引，所以虽然在age和register_time上均已经创建了索引，mysql查询优化器也只使用了age_idx索引，在Extra列中涉及的Using index condition指的是mysql先进行了age_idx索引条件过滤，而涉及的Using MRR指的是mysql进行索引过滤后取得一批id再回表查找。

创建了age和register_time的复合索引，笔者希望mysql在执行查询的时候首先会对age进行范围索引扫描，然后在register_time上进行范围索引扫描所以笔者将register_time索引作为复合索引的第二列索引.再次执行上述查询，发现mysql没有使用新创建的age_register_time_idx，笔者认为mysql查询优化器使用了错误的索引，于是笔者添加FORCE INDEX强制mysql使用age_register_time_idx索引。随后笔者再次执行查询，发现mysql执行仍然非常缓慢。使用Explain分析后发现Extra列有如下信息

```sql
Using where; Using index; 
```

Using index;说明了mysql使用了age_register_time_idx进行了条件过滤，但是Using where;说明mysql没有使用age_register_time_idx组合索引的register_time部分。查询资料后发现原因是mysql不支持松散索引扫描。也就无法实现从第一个范围age开始扫描到第一个范围age结束，然后扫描第二个register_time范围开始和第二个register_time范围结束。

### 优化方法一使用子查询

Mysql不支持松散索引扫描，每一个查询都只能使用一个索引进行范围扫描，那么我们是否可以将上述的查询拆分为两个子查询，其中一个子查询使用age索引进行范围扫描，而另一个子查询使用register_time索引进行范围扫描，然后取这两个子查询的交集id呢，然后在利用id回表查找用户信息。

```sql
SELECT count(*) 
FROM tb_user 
WHERE id IN (
    SELECT tb1.id 
    FROM
        ( SELECT id FROM tb_user FORCE INDEX ( `age_idx` ) WHERE age BETWEEN 18 AND 25 ) tb1
    INNER JOIN 
        ( SELECT id FROM tb_user FORCE INDEX ( `register_time_idx` ) WHERE register_time BETWEEN 20190101 AND 20191231 ) tb2 
    ON tb1.id = tb2.id 
)
```

发现这个查询性能非常慢，执行时长超过了30秒。这种优化方式失败，原因是因为满足条件的tb1和满足条件的tb2数据量都非常大，对这样的大的临时表取交集性能自然就非常的差。

### 优化方式二之使用散列值

Mysql不支持松散索引扫描，所以优化的思路是将多个范围查询优化为一个范围查询。对于上述这个例子来说，我们可以将age字段使用in来代替，如此便可以避免其中一个字段的范围查询

```sql
SELECT COUNT(*)
FROM tb_user 
WHERE age IN ( 18, 19, 20, 21, 22, 23, 24, 25 ) 
AND register_time BETWEEN 20190101 AND 20191231
```

优化之后，该查询使用了age_register_time_idx索引，该查询耗时为100ms左右。Explain分析如下

```text
+----+-------------+--------------+------------+-------+-----------------------------------------------------+------------------------+---------+------+-------+----------+--------------------------+
| id | select_type | table        | partitions | type  | possible_keys                                       | key                    | key_len | ref  | rows  | filtered | Extra                    |
+----+-------------+--------------+------------+-------+-----------------------------------------------------+------------------------+---------+------+-------+----------+--------------------------+
|  1 | SIMPLE      | tb_user      | NULL       | range | age_idx,register_time_idx,age_register_time_idx     | age_register_time_idx  | 8       | NULL | 16940 |   100.00 | Using where; Using index |
+----+-------------+--------------+------------+-------+-----------------------------------------------------+------------------------+---------+------+-------+----------+--------------------------+
```

上述的Explain分析有一个奇怪的地方在于Extra信息中涉及到where，按道理讲mysql count在索引就可以完成，没有必要回表，难道是mysql只使用了组合索引的前半部分age么？笔者强制上述查询使用age索引进行了验证，如果查询使用age索引和查询使用age_register_time_idx索引性能一样说明mysql只使用了组合索引age_register_time的前半部分。

```sql
mysql> SELECT
    -> count(*) 
    -> FROM
    -> tb_user FORCE INDEX(`age_idx`)
    -> WHERE
    -> age IN ( 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25 ) 
    ->  AND register_time BETWEEN 20190101  AND 20191231;
+----------+
| count(*) |
+----------+
|    16940 |
+----------+
1 row in set (6.76 sec)

mysql> SELECT
    -> count(*) 
    -> FROM
    -> tb_user FORCE INDEX(`age_register_time_idx`)
    -> WHERE
    -> age IN ( 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25 ) 
    ->  AND register_time BETWEEN 20190101  AND 20191231;
+----------+
| count(*) |
+----------+
|    16940 |
+----------+
1 row in set (0.01 sec)
```

### 优化方式三之使用冗余字段

冗余字段是解决慢查询的利器，上述的业务是查询2019注册的年轻用户的总数。我们也可以利用两个冗余字段，比如加入一个age_type字段，1代表年轻用户，2代表中年用户，3代表老年用户。同时创建一个age_type和register_time的复合索引，需要注意的是mysql不支持松散索引扫描，所以要将范围扫描的register_time字段放在组合索引的后半部分。查询语句如下所示

```sql
SELECT COUNT(*)
FROM tb_user 
WHERE age_type = 1
AND register_time BETWEEN 20190101 AND 20191231
```

虽然冗余字段为我们带来了便利，但是也为我们带来了管理上的麻烦。我们如何维护age_type字段呢？MySQL自带的触发器是一个比较好的方法，在age字段被更新的时候，触发器同时更新age_type字段。但是触发器可维护性比较差，我们也可以在业务层面手动维护age_type。这个业务需求实时性不是很高，我们可以开启一个异步线程每天凌晨扫描一遍表，更新错误的age_type字段。

## MYSQL中BETWEEN AND的使用对索引的影响

查询数据时，如果走普通索引，那么会产生回表操作，因为普通索引属于非聚集索引，叶子节点存放的是主键字段的值，拿到主键字段后再去表中根据主键值找到对应的记录。
因此，当数据量很大，而查询数据也很大时，考虑到回表的消耗，就不走索引；
当数据量很大，而查询数据很小，这个时候比起全表扫描，回表的消耗相对少，所以走索引

[相关链接](https://www.cnblogs.com/xinxinmifan/p/mysql_index_bewteen_and.html)

## 索引失效情况

[相关链接](https://www.jianshu.com/p/9c9a0057221f)

- 最好全值匹配——索引怎么建我怎么用
  
  - ![img](https://upload-images.jianshu.io/upload_images/6376767-407df3294e313dd7.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)
  
  在执行常量等值查询时，改版索引列的顺序并不会更改explan的执行结果，因为Mysql底层优化器会进行优化，但是推荐索引顺序列编写sql语句
  
- 最佳左前缀法则——如果索引了多列，要遵守最左前缀法则。指的是查询要从索引的最左前列开始并且不跳过索引中的列

  - mysql底层优化器会将sql按索引顺序进行优化

- 不在索引列上做任何操作（计算，函数，（自动或者手动）类型装换），会导致索引失效而导致全表扫描。——MYSQL自带api函数操作，如：left等

- 存储引擎不能使用索引中范围条件右边的列。——范围之后索引失效。（< ,> between and,）
  
  - ![img](https://upload-images.jianshu.io/upload_images/6376767-ab498edcf064ab61.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)
  
- 尽量使用覆盖索引（只访问索引的查询（索引和查询列一致）），减少select*。——按需取数据用多少取多少。

- 在MYSQL使用不等于（<,>,!=）的时候无法使用索引，会导致索引失效

- is null或者is not null 也会导致无法使用索引

- like以通配符开头（'%abc...'）MYSQL索引失效会变成全表扫描的操作。——覆盖索引
  
  - 在name和age上添加索引，查询，索引被使用。这样，单独查询name和单独查询age时都会使用到索引
  
- 字符串不加单引号索引失效

- 少用or，用它来连接时索引会失效

- 1、MySQL支持两种方式的排序filesort和index，Using index是指MySQL扫描索引本身完成排序。index效率高，filesort效率低。

  2、order by满足两种情况会使用Using index。


  ​	1.order by语句使用索引最左前列。

  ​	2.使用where子句与order by子句条件列组合满足索引最左前列。


  3、尽量在索引列上完成排序，遵循索引建立（索引创建的顺序）时的最佳左前缀法则。

  4、如果order by的条件不在索引列上，就会产生Using filesort。

  5、group by与order by很类似，其实质是先排序后分组，遵照索引创建顺序的最佳左前缀法则。注意where高于having，能写在where中的限定条件就不要去having限定了 作者：软件测试小将 https://www.bilibili.com/read/cv7624142/ 出处：bilibili

## [MySQL函数索引及优化](https://www.cnblogs.com/gjc592/p/13233377.html)

## IN和EXISTS优化

原则：小表驱动大表，即小的数据集驱动大的数据集

in：当B表的数据集必须小于A表的数据集时，in优于exists

```sql
SELECT * FROM A WHERE ID IN (SELECT ID FROM B)
```

exists：当A表的数据集小于B表的数据集时，exists优于in

```sql
SELECT * FROM A WHERE EXISTS (SELECT 1 FROM B WHERE B.ID = A.ID)
```

将主查询A的数据，放到子查询B中做条件验证，根据验证结果（true或false）来决定主查询的数据是否保留

1、EXISTS (subquery)只返回TRUE或FALSE,因此子查询中的SELECT * 也可以是SELECT 1或select X,官方说法是实际执行时会忽略SELECT清单,因此没有区别

2、EXISTS子查询的实际执行过程可能经过了优化而不是我们理解上的逐条对比

3、EXISTS子查询往往也可以用JOIN来代替，何种最优需要具体问题具体分析

## [MYSQL关于OR的索引问题](https://www.jianshu.com/p/6e90e38d2b67)

https://www.cnblogs.com/soysauce/p/10414296.html

## 大表删除总结

- 表数据量太大时，除了关注访问该表的响应时间外，还要关注对该表的维护成本（如做DDL表更时间太长，delete历史数据）
- 对大表进行DDL操作时，要考虑表的实际情况（如对该表的并发表，是否有外键）来选择合适的DDL变更方式
- 对大数据量表进行delete，用小批量删除的方式，减少对主实例的压力和主从延迟

## MySQL的一个表最多可以有多少个字段

总结

- [MySQL](https://cloud.tencent.com/product/cdb?from=10680) Server最多只允许4096个字段
- InnoDB 最多只能有1000个字段
- 字段长度加起来如果超过65535，My[SQL server](https://cloud.tencent.com/product/sqlserver?from=10680)层就会拒绝创建表
- 字段长度加起来（根据溢出页指针来计算字段长度，大于40的，溢出，只算40个字节）如果超过8126，InnoDB拒绝创建表
- 表结构中根据Innodb的ROW_FORMAT的存储格式确定行内保留的字节数（20 VS 768），最终确定一行数据是否小于8126，如果大于8126，报错。

优化建议

- 放弃使用Antelope这种古老的存储格式吧，原因上面也说到了把大字段的前768字节放在数据页中，这样会导致索引的层级很高，会直接影响到查询的性能。
- 对于大字段类型建议单独存放到一张表中，不要与经常访问的表放在一起，会造成物理IO的增加。

三种报错的疑惑

https://cloud.tencent.com/developer/article/1072013

## 学习博客

http://www.gxitsky.com/type/20