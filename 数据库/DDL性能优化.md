

## 执行顺序

**from**

**on**

**join**

**where**

**group by**

**having**

**select**

**field**

**order by**

**limit**

## 方案

### 1、深分页

假设`limit 20000, 20`，MySQL会扫描前20000个数据，然后再从20001开始拿数据，再丢弃前20000条数据，因此还是会顺序扫前面的20000条数据，而不是跳到20001开始扫描的。

#### 子查询

```mysql
select * from table A t1 join (
	select id from table A 
    where ......
 	order by indexA des 
 	limit 20000, 20
) t2 on t1.id = t2.id
```

使用子查询只取id能让查询排序排序更快。

减少无效行的列

> TODO：in和join的性能

#### 阈值倒序

当页码大于某个阈值（比如总页数的一半）时，可将排序颠倒进行查询

#### 排序作条件区间

把上一页排序项的极值作为条件传入

```mysql
SELECT   *
FROM     table
WHERE    ......
AND      indexA > '100'
ORDER BY indexA limit 10;
```

有个小缺陷，indexA区分度要求极高，最好是唯一，不然会缺数据

### 2、JOIN

#### 缩小左右表

```mysql
select * from 
table_a a 
left join table_b b on a.id = b.id
where a.disabled = 0 and b.disabled = 0;
```

减少join时左右表的行数

```mysql
select * from 
(select * from table_a
        where disabled = 0) a 
left join (
        select * from table_b
        where disabled = 0) b
    on a.id = b.id
```



### 3、索引

#### 查询



#### 排序

针对排序字段，一般会补充索引来进行优化，因此多字段排序的话，尽量让每个排序字段都单独设置一个索引，因为索引已经帮我们做好排序了。

> 尽量不要接多字段同时排序的需求，这种情况下索引的设计将会十分复杂

### 单表拆分

如果将多个条件构造联合索引，需要满足最左匹配原则，此时联合索引的数量将会很大，届时索引树也会十分复杂。单表拆分，此时针对单表的排序字段建立对应的索引，且单表职责更加单一

- 查询某个字段的排序数据时，在MyBatis层面，根据排序字段，指定查询排序字段所对应的单表。
- 单表拆分后，需要合理创建索引

### 慢查询日志

介绍：MySQL的慢查询日志是MySQL提供的一种日志记录，他用来记录在MySQL中响应时间超过阀值的语句，具体指运行时间超过`long_query_time`值的SQL，则会被记录到慢查询日志中。可以由它来查看哪些SQL超出了我们最大忍耐时间值。

默认情况下，MySQL数据库没有开启慢查询日志，需要手动设置参数：

- 查看是否开启：`show variables like '%slow_query_log%';`
- 开启日志：`set global slow_query_log = 1;`
- 设置时间: `set global long_query_time = 1;`
- 查看时间: `SHOW VARIABLES LIKE 'long_query_time%';`
- 查看超时的sql记录日志：Mysql的数据文件夹下`/data/...-slow.log;`在Navicat中输入`show variables like '%slow_query_log%`命令'，就可以得到文件目录;

## 参考

[MySQL深分页 + 多字段排序场景的优化方案](https://mp.weixin.qq.com/s/nbdQiBc92vcKwHZNlYYDyg)

[8种专坑同事的 SQL 写法，性能降低100倍](https://mp.weixin.qq.com/s/qC7pgKW3y0Y3tiAYXXqI2g)

https://mp.weixin.qq.com/s/2sssv-VB4-TYZe-9FZRGtA