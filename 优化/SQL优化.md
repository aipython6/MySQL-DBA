# SQL优化

## 全局优化

涉及到整个系统的优化，外界环境对数据库的影响

### 全局优化工具

对于Oracle而言，整体的优化工具主要有：

- *AWR：关注数据库的整体性能报告
  - load profile
  - effiency percentages
  - top 5 timed events
  - SQL Statistics
  - segment_statistics
- ASH：数据库中的等待事件与哪些SQL具体对应的报告
  - 等待事件与SQL结合
- ADDM：Oracle给出的一些建议
  - 各种建议与对应的SQL
- AWRDD：Oracle针对不同时段的性能的一个对比报告
  - 不同时期的load profile的比较
  - 不同时期等待事件的比较
  - 不同时期TOP SQL的比较

**掌握以上几种报告的获取方式**

#### AWR关注点

DB Time

load_profile

efficiency percentages

top 5 events

SQL Statistics

Segment Statistics



## 局部优化

内部执行语句的优化，不关注外部的环境影响

### 局部优化工具

主要是通过分析SQL语句的执行计划，主要的工具：

- explain plan for
- set autotrace on

- statistics_level=all
- awrrp.sql
- 10046 trace
- 通过sql_id获取

