# Windows安装MySQL

## MSI安装

installer方式安装

1.安装完成后停止服务，修改目录和设置环境变量

2.准备my.ini文件参数

3.重新初始化数据库

```
mysqld --defaults-file=e:\mysql\data\my.ini --initialize --basedir=e:\mysql\mysql57 --datadir=e:\mysql\data\data
```

4.去错误日志查看初始密码

同时还要启动mysql服务

```
net start mysql57
```

5.修改密码、启用远程连接、创建测试用户和测试数据

```mysql
create user 'zhangsan'@'%'identified by '123456';
flush privileges;

grant all privileges on dbname.* to 'zhangsan'@'%' identified by '123456';

grant all privileges on dbname.* to 'zhangsan'@'localhost' identified by '123456';

grant all privileges on *.* to 'zhangsan'@'%' identified by '123456' with grant option;

-- 检查用户
select host,user from mysql.user;

-- 删除用户
delete user from mysql.user where user='zhangsan' and host='localhost';
flush privileges;

-- 修改密码的而另一种方法
use mysql;
update user set authentication_string='123456' where user='root';
flush privileges;
-- 或
set password=password('123456');
flush privileges;
```



## zip Archive安装

将.zip文件解压到e:/mysql/mysql57

配置环境变量

```
MYSQL_HOME=E:\mysql\mysql57
PATH=;%MYSQL_HOME\bin
```

初始化数据库

```
mysqld --defaults-file=e:\mysql\data\my.ini --initialize --basedir=e:\mysql\mysql57 --datadir=e:\mysql\data\data
```

安装mysql系统服务

```
mysqld.exe -install mysql5 --defaults-file=e:\mysql\data\my.ini --initialize --basedir=e:\mysql\mysql57 --datadir=e:\mysql\data\data
```









