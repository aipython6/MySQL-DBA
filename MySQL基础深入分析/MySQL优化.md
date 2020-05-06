# MySQL优化

## 查询优化器

MySQL内部的执行计划

## 常见瓶颈

1.CPU

CPU在饱和时，一般发生在数据装入内存或从磁盘上读取数据时

2.IO

当装入的数据远大于内存容量时

3.硬件性能瓶颈

top、free、iostat、vmstat来查看系统性能状态

## Explain分析

### 字段分析

1. ​    id

id的值分为三种情况

- 全部的id都相同：表的执行顺序是从上往下执行的，这样查看table列的信息，就可以知道查询中所有表的执行顺序

- 全部的id都不同：id值越大，那么优先级就越高，越先被执行
- 部分id相同：参考上面两种情况（id相同，可以作为一组，从上往下执行，id不同，id值越大，越先被执行）

2.    select_type

查询类型

- SIMPLE：简单的select查询，不包含子查询或UNION
- PRIMARY：查询中若包含任何复杂的子部分，最外层插叙被标记为PRIMARY
- SUBQUERY：在select或where中包含了子查询
- DERIVED：在FROM列表中包含了子查询，则被标记为DERIVED，在执行递归查询过程中会执行这些子查询，把结果放在临时表中
- UNION：若第二个select出现在UNION之后，则被标记为UNION；若UNION包含在FROM字句的子查询中，外层的SELECT被标记为DERIVED
- UNION RESULT：从UNION表获取结果的SELECT

3.    table

查询中表的执行顺序，不能认为是从上到下的顺序，而是需要根据id的值是否相同来查看表的执行顺序

4.    type

type表示访问类型

const>eq_ref>ref>range>index>ALL

- ALL：全表扫描
- index
- range
- ref：当使用二级索引进行等值匹配时，对表的访问方法就是ref

```mysql
explain select * from t6 where name='a';
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | t6    | NULL       | ref  | idx_name      | idx_name | 17      | const |    1 |   100.00 | Using index |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
```



- eq_ref：用于联表查询中的唯一性索引扫描（即可以通过被驱动表的主键索引或唯一二级索引来进行等值匹配）

```mysql
explain select * from t5,t6 where t5.id=t6.id;
+----+-------------+-------+------------+--------+---------------+---------+---------+------------+------+----------+-------+
| id | select_type | table | partitions | type   | possible_keys | key     | key_len | ref        | rows | filtered | Extra |
+----+-------------+-------+------------+--------+---------------+---------+---------+------------+------+----------+-------+
|  1 | SIMPLE      | t6    | NULL       | ALL    | PRIMARY       | NULL    | NULL    | NULL       |    3 |   100.00 | NULL  |
|  1 | SIMPLE      | t5    | NULL       | eq_ref | PRIMARY       | PRIMARY | 4       | test.t6.id |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+--------+---------------+---------+---------+------------+------+----------+-------+
```



- const：精确匹配，只返回一条记录（用于主键索引或唯一的二级索引）
- NULL

5.   possible_keys



6.    key



7.    key_len



8.   ref



9.    rows



10.    Extra





## 索引优化











