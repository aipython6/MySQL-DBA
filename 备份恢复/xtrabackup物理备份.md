# xtrabackup物理备份

## 概述

### xtrabackup备份的原理



### xtrabackup备份的三个阶段

1.备份阶段

2.准备阶段

3.恢复阶段

## 1.xtrabackup全量备份备份

1.配置好my.cnf

2.创建备份目录

```shell
mkdir -p /mysql/backup
chown -R mysql:mysql /mysql/backup
```

3.备份阶段(生产环境建议创建一个备份用户)

```shell
innobackupex --defaults-file=/mysql/data/3306/my.cnf --user=root --password=123456 --no-timestamp /mysql/backup/fullback0426 --parallel=2 2>fullback0426.log
```

上面的是标准做法，即不使用默认的时间戳，并将日志记录到fullback0426.log文件中

下面的是一般做法

```shell
innobackupex --defaults-file=/mysql/data/3306/my.cnf --user=root --password=123456 /mysql/backup
```

4.准备阶段（目的是操作redo log）

操作之前先停止mysql服务，并创建好数据目录

```shell
service mysql stop
cd /mysql/data/3306/
mv data data_bak
mkdir data
chown -R mysql:mysql data
```

开始做准备阶段的操作

```shell
innobackupex --apply-log --user-memory=1G /mysql/backuo/fullback0426 
```

5.恢复阶段

```shell
innobackupex --defaults-file=/mysql/data/3306/my.cnf --copy-back /mysql/backup/fullback0426 --parallel=2

chown -R mysql:mysql data	#(该操作将xtrabackup用户改为mysql权限)

mysqld_safe --defaults-file=/mysql/data/3306/my.cnf &

#后面的步骤就是验证数据
```



## 2.xtrabackup备份与恢复部分数据库

1.备份部分数据库

```shell
innobackupex --defaults-file=/mysql/data/3306/my.cnf --user=root --password=123456 --no-timestamp /mysql/backup/partback01 --databases="test1 test2" --parallel=3
```

2.准备阶段

```shell
innobackupex --apply-log /mysql/backup/partback01/
```

3.关闭mysql服务，并保证test1和test2和ibdata*两个数据库不存在

```shell
service mysql stop

rm -rf ../data/test1
rm -rf ../data/test2
rm -rf ../data/ibdata*
```

4.将partback01目录中的test1、test2和iddata*全部拷贝到../3306/data目录下

```shell
cp -r /mysql/backup/partback01/test1/ /mysql/data/3306/data/
cp -r /mysql/backup/partback01/test2/ /mysql/data/3306/data/
cp -r /mysql/backup/partback01/ibdata* /mysql/data/3306/data/
chown -R mysql:mysql /mysql/data/3306/data/

#重启mysql，并检查数据
mysqld_safe --defaults-file=/mysql/data/3306/my.cnf
```

## 3.xtrabackup备份恢复-表

1.表的备份阶段

```shell
innobackupex --defaults-file=/mysql/data/3306/my.cnf --user=root --password=123456 --no-timestamp /mysql/backup/partback01 --databases="test1.per1 test2.per1" --parallel=2
```

也可以将要备份的表写入一个txt文件中，然后使用--tables-file来引入

````shell
vim tname.txt
test1.per1
test2.per1

#备份
innobackupex --defaults-file=/mysql/data/3306/my.cnf --user=root --password=123456 --no-timestamp /mysql/backup/partback01 --tables-file=../tname.txt --parallel=2
````

2.准备阶段

```shell
innobackupex --apply-log /mysql/backup/partback01
```

3.停止mysql服务，保证test1和test2数据库中的per1.*的文件都不存在（有就先删除）

```shell
service mysql stop

#分别拷贝数据文件到../data/test1{test2}目录下
cp -r /mysql/backup/partback01/test1/per1.* /mysql/data/3306/data/test1/
cp -r /mysql/backup/partback01/test2/per1.* /mysql/data/3306/data/test2/

cp -r /mysql/backup/partback01/ibdata* /mysql/data/3306/data/

chown -R mysql:mysql /mysql/data/3306/data/

#重启mysql并验证数据
```

## 4.xtrabackup增量备份

在进行增量备份前，需要进行一次全备

1.全备

```shell
rm -rf /mysql/backup/fullback0426

innobackupex --defaults-file=/mysql/data/3306/my.cnf --user=root --password=123456 --no-timestamp /mysql/backup/fullback0426 --parallel=2
```

2.基于全备做增备

```mysql
#插入一些数据模拟产生事务
insert into test1 (id,name) values (3,'aaaaa');
insert into test1 (id,name) values (4,'bbbbb');
commit;
##做一次增备
innobackupex --defaults-file=/mysql/data/3306/my.cnf --user=root --password=123456 --no-timestamp --incremental /mysql/backup/incback01 --incremental-basedir=/mysql/backup/fullback0426 --parallel=2

#接着再产生一些事务
insert into test1 (id,name) values (5,'ccccc');
insert into test1 (id,name) values (6,'ddddd');
commit;
##再做一次增备
innobackupex --defaults-file=/mysql/data/3306/my.cnf --user=root --password=123456 --no-timestamp --incremental /mysql/backup/incback02 --incremental-basedir=/mysql/backup/incback01 --parallel=2
```

3.准备阶段

```shell
innobackupex --apply-log --redo-only /mysql/backup/fullback0426/

innobackupex --apply-log --redo-only /mysql/backup/fullback0426/ --incremental-dir=/mysql/backup/incback01

innobackupex --apply-log /mysql/backup/fullback0426/ --incremental-dir=/mysql/backup/incback02

innobackupex --apply-log /mysql/backup/fullback0426/

##注意，如果上面的过程报错，很有可能是因为LSN的位置没有确定准备，如果是这样，可以删除所有的备份文件，重新开始备份（但是需要重新模拟事务）


```

3.确保../data目录为空，且用户权限为mysql

```shell
#恢复
innobackupex --defaults-file=/mysql/data/3306/my.cnf --copy-back --rsync /mysql/backup/fullback0426/ --parallel=2

#重启数据库验证数据
```























































