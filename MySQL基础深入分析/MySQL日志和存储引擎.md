# MySQL文件、日志分析、字符集、用户管理、审计和存储引擎

## MySQL日志类型

### error log

mysql启动/运行/关闭过程的记录，记录错误/警告/正常的信息

```mysql
show variables like '%log_error%';

查看系统异常：/var/log/messages

log_error_verbosity:
只记录错误日志
记录错误+警告信息
记录错误/警告/正常，默认值为3
```



### general log

通用日志：记录了mysql数据库请求的信息

```mysql
mysql> show variables like '%general_log%';
+------------------+------------------------+
| Variable_name    | Value                  |
+------------------+------------------------+
| general_log      | OFF                    |				（默认是关闭）
| general_log_file | /var/lib/mysql/aaa.log |
+------------------+------------------------+

临时开启：
set @@global.general_log=on;
```

平常主要用例当成yingxiang数据库的案例审计，查询做了什么事情。但是生产环境不建议开，因为日志太多，虽然安全，但是性能会受到影响



### binary log

记录数据库发生更改的sql语句，以二进制形式保存在磁盘中

作用：恢复、复制、审计

特点：

1.以sql语句的形式记录

2.在commit才会写binlog，在此之前都是写日志缓存（binlog_buffer），提交时从binlog_buffer写回磁盘的binlog中。binlog不会被覆盖，是一直存在的（当满了之后，会重新生成一个文件继续保存新的日志）

3.可以做备份后的恢复

4.对所有表起作用



查看binlog是否开启（如果数据不是特别重要，建议不要开启，但是为了安全，最好是开启binlog）：

```mysql
show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000240 |     1737 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+


show variables like "%log_bin%";
show variables like '%binlog%';

-- 开启binlog，修改my.cnf
log_bin=/mysql/log/3306/binlog/bin-log		（从bin-log.00001开始）
log_bin_index=/mysql/log/binlog/bin-log.index
注意：需要修改相应目录的权限给mysql用户
mkdir -p /mysql/log/3306/binlog/
chown -R mysql:mysql /mysql/log/3306/binlog/
chown -R 775 /mysql/log/3306/binlog/
再重启mysql

```



### relay log

主要用于主从复制

用于存储从服务器的IO线程接收来自主服务器发来的变更日志

show variables like '%relay%';

### slow log

跟SQL优化相关

记录SQL查询慢的日志，相关参数

slow_query_log

slow_query_log_file

slow_query_time

```mysql
show variables like 'slow_query_log';


mysql> show variables like '%_query%';
+------------------------------+-----------------------------+
| Variable_name                | Value                       |
+------------------------------+-----------------------------+
| binlog_rows_query_log_events | OFF                         |
| ft_query_expansion_limit     | 20                          |
| have_query_cache             | YES                         |
| long_query_time              | 10.000000                   |			(大于10秒就会记录慢查询日志)
| slow_query_log               | OFF                         |
| slow_query_log_file          | /var/lib/mysql/aaa-slow.log |
+------------------------------+-----------------------------+

```

慢查询的原因：

1.lock_time锁等待的时间太长

2.examined处理的数据太多

```mysql
show variables like '%using_indexes%';
+----------------------------------------+-------+
| Variable_name                          | Value |
+----------------------------------------+-------+
| log_queries_not_using_indexes          | OFF   |		（该参数表示如果没有使用索引，也会记录到慢查询日志中，所以最好是打开）
| log_throttle_queries_not_using_indexes | 0     |		（值为0，表示每分钟都对慢查询日志进行刷新，所以最好设置为10分钟，表示未使用使用索引的sql语句次数）
+----------------------------------------+-------+

查看某条语句是否使用索引

mysql> explain select * from employees;
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299202 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------+
1 row in set, 1 warning (0.01 sec)

possible_keys为NULL，表示没有使用索引

mysql> explain select * from employees where emp_no=10001;
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | employees | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+-----------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

possible_keys为PRIMARY表示使用了主键索引
```

慢查询分析工具：mysqldumpslow

```shell
mysqldumpslow /var/lib/mysql/aaa-slow.log
```

mysqldumpslow查看常用参数

```
-s ORDER     what to sort by (al, at, ar, c, l, r, t), 'at' is default		（按照什么方式排序）
                al: average lock time		平均锁时间
                ar: average rows sent		平均返回记录
                at: average query time		平均查询时间
                 c: count			（记录次数）
                 l: lock time		锁的时间
                 r: rows sent	返回记录
                 t: query time  	查询时间
                 
-t NUM       just show the top n queries	top N行，返回前面有多少行数据
-n NUM       abstract numbers with at least n digits within names		返回至少n条数据
-g PATTERN   grep: only consider stmts that include this string		正则匹配模式，不区分大小写显示结果

例如：
1询按平均锁时间，返回前10条
mysqldumpslow -s al -n 10 /var/lib/mysql/aaa-slow.log

2.获取慢查询日志文件中按查询时间排序的前10条里面包含有左连接的慢查询语句
mysqldumpslow -s t -t 10 -g "left join" /var/lib/mysql/aaa-slow.log

```

log_out：输出的文件格式

```mysql
mysql> show variables like '%log_out%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_output    | FILE  |
+---------------+-------+

建议为FILE的形式，即保存在文件中
```

注意：正在慢的SQL不能查询，只有执行完成后才可以查询

### redo log

记录的是对物理页的修改，记录了关于DML的操作，redo log是循环覆盖的，它能保证脏页没有写到磁盘上时，对应的redo log是不会被覆盖的，它用于数据库的崩溃恢复，只针对InnoDB的表起作用。

它先写log buffer，最后再刷到磁盘中

### undo log





## 其他文件

### socket

IP+端口

进行网络通信必须的5种信息：协议、本地IP、本地协议端口、远程IP、远程协议端口

```mysql
show variables like '%socket%';
```

如果主机上有多个实例，通过socket可以连接到相应的实例

mysql -uroot -p -S /mysql/data/3306/mysql.sock

### pid文件

记录的是mysqld的进程号，每次开机都会写入当前的pid

```mysql
show variables like '%pid%';

```

### 表的结构文件

跟存储引擎有关

```
InnoDB：
.frm：为表结构文件，记录表的结构定义
.idb：表的数据和索引信息
```



### InnoDB存储引擎相关的文件

表空间文件：数据文件、临时文件

```mysql
show variables like 'innodb_file_per_table';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_file_per_table | ON    |		
+-----------------------+-------+

(innodb_file_per_table：该参数分为共享表空间和独立表空间，OFF/0：共享表空间（所有的数据和索引全部放在一个文件），ON/1：表示独立表空间（每个表的数据和索引存在各自的表空间中，建议使用该模式））
 
 show variables like '%innodb%data%';
 +----------------------------+------------------------+
| Variable_name              | Value                  |
+----------------------------+------------------------+
| innodb_data_file_path      | ibdata1:12M:autoextend |			
| innodb_data_home_dir       |                        |
| innodb_stats_on_metadata   | OFF                    |
| innodb_temp_data_file_path | ibtmp1:12M:autoextend  |
+----------------------------+------------------------+
( innodb_data_file_path：表示如果设置了datadir目录，就保存在该目录下，在实际生产中，可以设置得更大，如5G等，该参数只能在my.cnf中进行修改)
 
 show variables like 'datadir';
 
```



redo log

undo log

```mysql
mysql>  show variables like '%undo%';
+--------------------------+------------+
| Variable_name            | Value      |
+--------------------------+------------+
| innodb_max_undo_log_size | 1073741824 |
| innodb_undo_directory    | ./         |
| innodb_undo_log_truncate | OFF        |
| innodb_undo_logs         | 128        |
| innodb_undo_tablespaces  | 0          |
+--------------------------+------------+
```



## MySQL日志分析工具

### mysqldumpslow

mysql官方提供的慢查询日志分析工具

主要功能是统计不同慢sql的出现次数（count）、执行耗费的平均时间和累计总耗时时间（time）、等待锁耗费的时间（lock）、发送给客户端的总行数（rows）、扫面的总行数（rows）、sql语句本身

### mysqlbinlog

官方提供的日志分析工具

### pt-query-digest（4.2.9节介绍）

用于分析慢查询的一个工具，可以分析binlog、general log、slow log，也可以通过show processlist或tcpdump抓取MySQL协议数据来进行分析。可以把分析结果输出到文件中，分析过程是先对查询的条件进行参数化，然后对参数化以后的慢查询进行分组统计，统计出各查询的执行时间、次数、占比等，可以借助分析结果找出问题并进行优化。

```
案例
1.分析最近12小时内的查询：
pt-query-digest --since=12 /mysql/log/3306/query.log > slow.log


```



### mysqlsla（4.2.8节介绍）

mysqlsla.com推出的一款日志分析工具，输出的数据报表非常有利于分析慢查询的原因，包括执行频率、数据量和查询消耗等，功能非常强大。

mysqlsla可以解决的问题：分析mysql的所有日志，general log、slow log、binlog

处理流程：加载日志--解析日志--过滤日志--排序--出报告--重演

核心功能：过滤日志、出报告

### mysql-log-filter

简洁报表，推荐使用一下



## MySQL的默认数据库

### information_schema

提供数据库的元数据，如数据库的数据名、表名、列信息、访问权限、索引、视图、存储过程、函数等信息

### performance_schema

收集数据库服务器性能参数，得到数据库的运行统计信息

### mysql

是mysql的核心数据库，类似于sql-server的master库和oracle的system。主要负责存储数据库的用户\权限等mysql自己需要使用的控制权限和管理信息.保存了用户信息、权限信息、存储过程、event和时间信息等.

（不能随便删除和修改）

```
+---------------------------+
| Tables_in_mysql           |
+---------------------------+
| columns_priv              |
| db                        |
| engine_cost               |
| event                     |			事件和任务调度
| func                      |				函数
| general_log               |		
| gtid_executed             |	mysql启动阶段会从这个表来获取gtid里面的变量值
| help_category             |
| help_keyword              |
| help_relation             |
| help_topic                |
| innodb_index_stats        |		InnoDB索引统计信息
| innodb_table_stats        |		InnoDB表的统计信息
| ndb_binlog_index          |		NDB集群二进制日志索引信息
| plugin                    |						插件表
| proc                      |							
| procs_priv                |
| proxies_priv              |
| server_cost               |
| servers                   |
| slave_master_info         |
| slave_relay_log_info      |
| slave_worker_info         |
| slow_log                  |
| tables_priv               |
| time_zone                 |
| time_zone_leap_second     |
| time_zone_name            |
| time_zone_transition      |
| time_zone_transition_type |
| user                      |
+---------------------------+

1、成本模型
engine_cost:IO
server_cost :CPU

2、权限相关的表：
columns_priv：列级别的权限
db：库级别权限
user：用户账号，全局权限
tables_priv：表级别权限
procs_priv：存储过程和函数的相关权限
proxies_priv：代理用户的权限
```



### sys

它的数据来源于performance_schema



### 其他常用命令

```mysql
-- 查看复制容灾：
show master status;
show slave status;

-- 查看数据库统计状态
show status;

-- 查看单机还是集群
show variables like '%cluster%';


-- 触发器和存储过程
show triggers;
show procedure status;

-- mysql线程
show processlist;

-- 查看用户的授权信息
show grants for user_name;

```



## MySQL的字符集管理

### 字符集和校验规则

是一套字符与字符编码的集合，用于显示一些抽象的符号

校验规则：字符集排序规则

#### 常见的字符编码

ASCII、gb2312、gbk、latin1、unicode

```mysql
-- 查看服务器支持的字符集
show character set;

-- 查看字符集的校验规则
show collation;

utf8：一个中文字符是3个字节
utf8mb4：一个中文字符是4个字节

-- 查看当前数据库字符集
show variables like 'character%';

```



### 字符集的设置

#### 服务器级别

```
编译时设置
配置文件my.cnf指定
[mysqld]
character_set_server=utf8
-- skip-character-set-client-handshake=1	客户端按服务器端字符集


环境变量中指定
export LANG=en_US.UTF8

启动时指定：
mysqld --character-set-server=utf8 &

客户端连接时指定：
mysql -uroot -p --character-set-server=utf8

临时统一指定：
set names utf8;


临时更改：
set global character-set-server=utf8;

永久修改：my.cnf，重启生效
```



#### 数据库级别

```mysql
create database db_name charset=utf8;

-- 字句
default charset=utf8
charset=utf8
character set utf8;

show create db_name;

```



#### 表级别/列级别

```mysql
create table tb_name()engine=innodb default charset=utf8;


alter table tb_name default charset utf8;
```



## MySQL权限管理

数据库安全分为：内部和外部

设计权限的SQL

```mysql
DDL：
create
drop
alter
truncate

DML：
insert
update
delete

DQL：
select

DCL：
grant
revoke

TCL：
commit
rollback
```

用户权限管理的作用：

- 限制用户访问哪些库/表
- 限制用户对哪些表执行select、create、delete、drop、alter等操作
- 限制用户登录的IP或域名
- 限制用户自己是否有权限授权给其他用户

用户分为：普通用户、root用户（超级管理员）

```mysql
-- 权限管理帮助
help account management;

   ALTER USER
   CREATE USER
   DROP USER
   GRANT
   RENAME USER
   REVOKE
   SET PASSWORD

```

用户权限管理的表：

- user

查看全局所有数据库的权限

```mysql
mysql> select host,user from user;
+-------------+------------------+
| host        | user             |
+-------------+------------------+
| %           | root             |					（%表示所有主机）
| 10.42.0.184 | root             |
| localhost   | debian-sys-maint |		（localhost表示本地主机）
| localhost   | mysql.session    |
| localhost   | mysql.sys        |
| localhost   | root             |
| localhost   | zhangsan         |
+-------------+------------------+

select * from user;
返回的结果分为4类：用户列、权限列、安全列、资源控制列


```



- db

存储用户对某个数据库的操作权限，决定用户从哪个主机访问哪个数据库

```mysql
select * from db;
```



- tables_priv

用于对表设置操作权限

```mysql
select * from tables_priv;
+-----------+---------+---------------+------------+----------------------+---------------------+------------+-------------+
| Host      | Db      | User          | Table_name | Grantor              | Timestamp           | Table_priv | Column_priv |
+-----------+---------+---------------+------------+----------------------+---------------------+------------+-------------+
| localhost | mysql   | mysql.session | user       | boot@connecting host | 0000-00-00 00:00:00 | Select     |             |
| localhost | sys     | mysql.sys     | sys_config | root@localhost       | 2020-02-01 07:10:46 | Select     |             |
| localhost | MyTest1 | zhangsan      | P          | root@localhost       | 0000-00-00 00:00:00 | Update     |             |
+-----------+---------+---------------+------------+----------------------+---------------------+------------+-------------+

```



- columns_priv

用于对表的某一列设置操作权限

- proces_priv

用来对存储过程和存储函数设置操作权限

- proxies_priv

代理用户/角色权限

### MySQL权限管理方式

mysql的权限管理是：用户+IP

如：

user1@127.0.0.1

user1@localhost

user1@192.168.1.2

上面同一个用户对应的是不同的权限，因为它们的IP不同

#### 权限范围分为三类

- SQL语句
- 存储过程、触发器
- 管理类权限

#### 显示权限

```mysql
show grants for 用户名@IP

-- 如
show grants for zhangsan@localhost;
+---------------------------------------------------------+
| Grants for zhangsan@localhost                           |
+---------------------------------------------------------+
| GRANT USAGE ON *.* TO 'zhangsan'@'localhost'            |
| GRANT UPDATE ON `MyTest1`.`P` TO 'zhangsan'@'localhost' |
+---------------------------------------------------------+

-- 创建用户语法
CREATE USER [IF NOT EXISTS]
    user [auth_option] [, user [auth_option]] ...
    [REQUIRE {NONE | tls_option [[AND] tls_option] ...}]
    [WITH resource_option [resource_option] ...]
    [password_option | lock_option] ...

user:
    (see )

auth_option: {
    IDENTIFIED BY 'auth_string'
  | IDENTIFIED WITH auth_plugin
  | IDENTIFIED WITH auth_plugin BY 'auth_string'
  | IDENTIFIED WITH auth_plugin AS 'hash_string'
  | IDENTIFIED BY PASSWORD 'hash_string'
}

tls_option: {
   SSL
 | X509
 | CIPHER 'cipher'
 | ISSUER 'issuer'
 | SUBJECT 'subject'
}

resource_option: {
    MAX_QUERIES_PER_HOUR count
  | MAX_UPDATES_PER_HOUR count
  | MAX_CONNECTIONS_PER_HOUR count
  | MAX_USER_CONNECTIONS count
}

password_option: {
    PASSWORD EXPIRE
  | PASSWORD EXPIRE DEFAULT
  | PASSWORD EXPIRE NEVER
  | PASSWORD EXPIRE INTERVAL N DAY
}

lock_option: {
    ACCOUNT LOCK
  | ACCOUNT UNLOCK
}

-- 例如，下面表示两个用户，因为他们的主机位置不同，但是这两个用户是没有权限的，只能登录
create user zhangsan@localhost;		-- 创建没有密码的用户，localhost代表只有本机可以访问
create user zhangsan@'%'	identified by '123456';		--  创建有密码的用户%表示代表所有主机都能使用该用户访问

-- 上面第一种登录方法：
mysql -uzhangsan -hlocalhost -p

-- 上面第二种登录方法：
mysql -uzhangsan -h192.168.1.2 -p

-- 授权(all:表示所有权限，*.*：表示所有数据库中的所有表，grant option：表示该用户可以将权限授权给其他用户)
show all privileges on *.* to 'wangwu'@'%' identified by '123' with grant option;

-- 授权范围
on *.*			保存在mysql.user
on 库名.*		mysql.db
on 库名.表名		mysql.tables_priv
on 库名.表名.列名			mysql.columns_priv
on 库名.存储过程/函数		mysql.proces_priv

select * from mysql.user where user='zhangsan' and host='localhost';

select current_user();

-- 刷新权限
flush privileges;

grant select,insert,update,delete on *.* to zhangsan@'%' ;


```

#### 开发人员授权

（创建、删除、修改）表/索引/视图/存储过程/函数等权限

```mysql
create user dev@'%'	identified by '123456';		-- 一般是授权给外面机器的用户登录，而不用本机登录
grant create,alter,drop on db_name.* to dev@'%';
flush privileges;


-- 创建指定网段可以访问数据库的用户
create user dev@'192.168.1.%' identified by '123456';
grant create,alter,drop on db_name.* to dev@'192.168.1.%';

-- 操作外键
grant references on db_name.* to dev@'%';

-- 创建临时表
grant create  temporary tables on db_name.* to dev@'%';

-- 操作索引
grant index db_name.* to dev@'%';
flush privileges;

-- 操作视图
grant create view,show view on db_name.* to dev@'%';\
flush privileges;

-- 操作存储过程和函数
grant create routine,alter routine,execute on db_name.* to dev@'%';
flush privileges;

-- 授权一个普通的dba用户，可以管理某一个数据库
grant all privileges on db_name.* to dev@'localhost' identified by '123456';
flush privileges;

-- 授权一个高级通的dba用户，可以管理所有数据库
grant all privileges on *.* to alldba@'localhost' identified by '123456';
flush privileges;

-- 授权执行存储过程和函数
grant execute on procedure db_name.procedure_name to alldba@'localhost';
grant execute on function db_name.function_name to alldba@'localhost';
flush privileges;

```

#### 回收权限

针对所有库/表/存储过程

```mysql
REVOKE
    priv_type [(column_list)]
      [, priv_type [(column_list)]] ...
    ON [object_type] priv_level
    FROM user [, user] ...

REVOKE ALL [PRIVILEGES], GRANT OPTION
    FROM user [, user] ...

REVOKE PROXY ON user
    FROM user [, user] ...

-- revoke只回收权限，不删除用户

revoke execute on procedure db_name.procedure_name from alldba@'localhost';
revoke execute on function db_name.function_name from alldba@'localhost';
flush privileges;


revoke all privileges on db_name.* from dev@'localhost';
flush privileges;


-- 删除用户,删除用户后，权限也就直接收回
delete from mysql.user where user='zhangsan' and host='localhost';

drop user dev;
drop user dev@'%';
drop user dev@'192.168.1.%';


-- 重新命名用户
rename user 'zhangsan'@'localhost' to 'lisi'@'localhost';


```



### 用户密码管理

#### 修改用户密码

- 修改root密码

```mysql


1、mysqladmin -uroot -h localhost -p password 'new_password';

2、修改mysql.user表
use mysql;
update mysql.user set authentication_string=password('newpassword') where user ='root';
flush privileges;

3、用set语句
set password=password('newpassword');

4、使用alter user（MySQL5.7官方推荐）
alter user 'root'@'localhost' identified by 'newpassword';
```

- 修改普通用户密码

```mysql
1、修改mysql.user表
use mysql;
update mysql.user set authentication_string=password('newpassword') where user ='zhangsan' and host='localhost';
flush privileges;

2、使用grant（推荐使用），需要root登录
grant usage on *.* to 'zhangsan'@'%' identified by '123456';
grant usage on *.* to 'zhangsan'@'localhost' identified by '123456';
flush privileges;

3、当前用户登录
set password=password('newpassword');

4、使用alter user（MySQL5.7官方推荐）
alter user 'zhangsan'@'localhost' identified by 'newpassword';
alter user 'zhangsan'@'%' identified by 'newpassword';
```

#### 密码过期问题

mysql5.7.11之前有一个360天密码过期的问题

```mysql
-- 涉及到的密码过期参数
show variables like 'default_password_lifetime';
+---------------------------+-------+
| Variable_name             | Value |
+---------------------------+-------+
| default_password_lifetime | 0     |
+---------------------------+-------+
-- 1. 默认为0，永久不过期

设置my.cnf
[mysqld]
default_password_lifetime=0

-- 2.alter user
alter user 'zhangsan'@'localhost' password expire interval 90 day;
-- 查看
select * from mysql.user;

alter user 'zhangsan'@'localhost' password expire interval never;		-- 永久不过期
alter user 'zhangsan'@'localhost' password expire interval default;		-- 默认
```

#### 用户的锁定的解锁

```mysql
-- 锁定用户
alter user 'zhangsan'@'localhost' account lock;

-- 解锁用户
alter user 'zhangsan'@'localhost' account unlock;
```

#### root密码丢失的解决办法

Windows

```mysql
方法1、将--skip-grant-tables参数加入到my.ini文件中，重启MySQL server，登录时不再使用密码，进入后再修改密码(改完后删除该参数，重启生效)

方法2、mysqld --skip-grant-tables
use mysql;
update mysql.user set authentication_string=password('newpassword') where user ='root';
flush privileges;
```

Linux

```mysql
1.service mysql stop
2.加入
[mysqld]
--skip-grant-tables
到my.cnf中
3.service mysql start
4.mysql -uroot -p		登录不用密码
5.修改密码
use mysql;
update mysql.user set authentication_string=password('newpassword') where user ='root';
6.取消--skip-grant-tables参数
7.重启
8.登录
```

#### 常用等登录方式和免密登录的方法

```
---使用密码登录
①mysql -uroot -p
②mysql -p
③mysql -S /mysql/data/3306/mysql.sock -uroot -p
④mysql -h ip -uroot -p
⑤mysql -localhost -uroot -p123456


---免密登录
方法1：修改my.cnf
[client]
user="root"
password="123456"

此时登录方法
mysql
或
mysql --defsults-file=../my.cnf


方法2：不同客户端
[client]
user="root"
password="123456"

[mysqladmin]
user="root"
password="123456"


方法3：当前环境变量
vi ~/.my.cnf
[client]
user="root"
password="123456"
直接mysql登录


方法4：使用login-path（最安全的方法）
mysql_config_editor set --login-path=password --user=root --password
mysql_config_editor print --all

清除：
mysql --login-path=password
```



### 用户角色管理

角色：用于批量管理用户，同一个角色下面的数据都有相同的权限

mysql5.7是由mysql.proxies_priv来模拟实现的

```
参数：
check_proxy_users=on
mysql_native_password_proxy_users=on
```



## Federate远程访问数据库

### 概述

允许本地访问远程MySQL数据库中表的数据，本地只存结构，不存数据。本地的表结构相当于一个软连接，远程的数据库保存真正的数据

Federate存储引擎默认不开启，如果开启需要在my.cnf加入

```
[mysqld]
federate
```

mysql中的federate目前不支持异构数据库，但mariadb中的FederateX支持

show engines查看

注意：

- 本地表的结构必须与远程的完全一样
- 远程数据目前只支持mysql
- 不支持事务
- 不支持修改表的结构
- 一般只用于查询

### 如何使用Federate

#### 在建表语句末尾加入

```mysql
engine=federated connection='mysql://用户名:密码@IP:端口/数据库名/表名';
```

#### 使用

```
1、查看存储引擎
show engines;

2、配置启动federated
my.cnf中加入下面的参数并重启
[mysqld]
federated

service mysql restart
show engines;


```



## 审计

目的是用户查询证据，mysql5.7企业版自带审计功能，而社区版本可以使用开源版本——mysql Audit Pluging

另外还有一个审计工具：https://github.com/cookieY/Yearning

在生产环境中不建议开启，会影响性能，可以使用第三方审计



mysql自带的init-connect+binlog审计

my.cnf

init-connect

超级管理员的操作不会被记录到审计日志中

## MySQL安全SSL认证

### 概述

SSL安全通道，它的流程如下：

1.先为MySQL服务器创建SSL证书和私钥

2.在MySQL中配置SSL，并启动服务

3.创建用户时带上SSL标签

4.连接数据库时带上SSL

```mysql
show variables like '%ssl%';
+---------------+-----------------+
| Variable_name | Value           |
+---------------+-----------------+
| have_openssl  | YES             |
| have_ssl      | YES             |						(如果为disable，表示SSL没有启动)
| ssl_ca        | ca.pem          |
| ssl_capath    |                 |
| ssl_cert      | server-cert.pem |
| ssl_cipher    |                 |
| ssl_crl       |                 |
| ssl_crlpath   |                 |
| ssl_key       | server-key.pem  |
+---------------+-----------------+

```

### 配置SSL的方法

#### 手动配置

mysql5.7.6 -

#### 自动配置：

mysql5.7.6

创建并验证SSL的步骤：

1.openssl version

如果没有安装openssl，则需要先安装

yum install -y openssl

mysql_ssl_rsa_setup --datadir=/mysql/data/3306 --user=mysql --uid=mysql

会生成8个.pem文件



2.修改my.cnf，[mysqld]

ssl-ca=/mysql/data/3306/ca.pem

ssl-cert=/mysql/data/3306/server-cert.ca

ssl-key=/mysql/data/3306/server-key.pem



3.重启服务器

mysql service restart



4.检查状态

```mysql
show global variables like '%ssl%';
show global variables like 'tls_version';
```



5.配置SSL用户并测试

```MySQL
如果不设置必要的参数时，下面这种方法不强制用户认证SSL
方法1登录时使用：
mysql -uroot -p -- ssl-mode=require		使用
mysql -uroot -p -- ssl-mode=disable		禁用

使用status命令查看是否已经启用


方法2：
mysql -uroot -p --ssl-ca=/mysql/data/3306/ca.pem --ssl-cert=/mysql/data/3306/server-cert.ca --ssl-key=/mysql/data/3306/server-key.pem
可以使用这种方法进行远程连接
------------------------------
用户登录时需要强制认证，创建用户时加上require x509
create user 'zhangsan'@'%' identified by '123456' require x509;
grant all on *.* to 'zhangsan'@'%';
flush privileges;

```

取消用户认证

```mysql
alter user 'zhangsan'@'%' require none;
```

JDBC客户如何连接：在URL中加入ssl=true就可以开启SSL

## 存储引擎

### 概述

存储引擎就是各种类型的表

mysql中用得较多的存储引擎是：InnoDB（5.5以后默认）、MyISAM（8.0已经废弃）、MEMORY

### 分类

#### MyISAM

不支持事务、不支持行级锁、只支持表锁、主要用于高负载查询

支持3种不同的存储形式：静态表、动态表、压缩表

#### InnoDB

提供了事务功能（事务的提交、回滚和恢复等），支持行级锁，支持外键约束，支持自动增长列，使用B+Tree索引，叶子节点直接保存的是数据。

相比较于MyISAM，写的效率会差一些，并且会占用更多的磁盘空间以保存数据和索引.

#### 分析InnoDB

缓冲池默认16K，额外内存池默认值为8M

redo log buffer:重做日志信息--redo log buffer--重做日志文件

innodb_log_buffer_size的大小默认为8M

将重做日志缓冲中的内容刷新到外部磁盘的重做日志文件中的3种情况：

- master thread每一秒将重做日志缓冲刷新到重做日志文件
- 每个事务提交时会将重做日志缓冲刷新到重做日志文件中
- 当重做日志缓冲池剩余空间小于1/2时，重做日志缓冲刷新到重做日志文件中

#### MEMORY

使用存在于内存中的内容创建表，使用的是Hash索引，所以比B+Tree要快，但是也会带来很多问题，如不支持范围查询.

每个memory表只对应一个磁盘文件，格式为.frm，该文件只存储表的结构，而其数据文件都是存储在内存中，这样有利于对数据进行快速处理，提高整个表的处理能力。因为memory的数据是存储在内存中的，所以一旦服务管理，表中的数据就会丢失。

MEMORY默认使用hash索引，其速度比B+树快，但是它不支持范围查询，只支持=,<>

















