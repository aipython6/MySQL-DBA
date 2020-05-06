# Linux平台安装MySQL

只记录了关键步骤

## 二进制安装

1.	查看系统是否已经安装了其他版本的MySQL

```shell
rpm -qa | grep mysql

#如果有的话需要卸载
rpm -e 包名
```

2.my.cnf的默认文件路径

```shell
/etc/my.cnf

#可以删除这个默认的参数文件，然后新建一个my.cnf
/mysql/data/3306/my.cnf
```

3.初始化数据库

```shell
mysqld --defaults-file=../my.cnf --initliaze --user=mysql --basedir=../mysql --datadir=../data
```

4.配置service mysql start/stop/restart命令

```shell
cd ../support-files
cp mysql.server mysql

配置mysql文件
指定好之前配置的basedir、datadir路径

并在启动的地方加上 --defaults-file=../my.cnf

cp mysql /etc/init.d/
```

可以将mysql_safe做成命令放到后台启动

```shell
vim msyql.start
#插入如下内容
/mysql/app/mysql/bin/mysql_safe --defaults-file=/mysql/data/3306/my.cnf --user=mysql &

chmod u+x mysql.start

#通过这种方法使得一台机器可以启动多个mysql实例
```

停止mysql服务的一种方法

```shell
mysqladmin -uroot -p shutdown -S /mysql/data/3306/mysql.sock
```

处理密码过期的问题

```shell
#在my.cnf中加上
skip-grant-tables

mysql -uroot -p
#如果找不到/tmp/mysql.sock
可以使用如下的方法连接sock文件
ln -s /mysql/data/3306/mysql.sock /tmp/mysql.sock

#mysql中的操作
use mysql;
select * from mysql.user where user='root'\G

#将password-expired设置为不过期
update user set passowrd_expired='N' where user='root';
flush privileges;

最后需要把my.cnf中的skip-grant-tables删除
```

## 源码安装

1.检查环境需要的包和工具

```shell
mount /dev/cdrom /mnt
yum -y install gcc/g++ bison ncurses ncurses-devel zlib libxml2 openssl dtrace cmake libstdc++-devel gcc-c++

#添加mysql/bin到~/.bash_profile

#还需要先安装源码编译工具cmake，cmake需要去官方网站下载
#cmake具体的安装方法需要官网查看
#第一步
./bootstrap

#第二步使用gmake编译
gmake

#第三步安装
gmake install

```

2.创建用户，组，创建目录,改权限

```shell
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
mkdir -p /mysql/data/3306/data
mkdir -p /mysql/log/3306
chown -R mysql:mysql /mysql
```

3.查看系统是否已经预安装mysql

```shell
rpm -qa |grep mysql

#如果有就需要卸载
rpm -e 包名
```

4.编译安装

```shell
#先解压源码包
tar -xzvf ../tar.gz
cd /mysql/app/mysql-5.7.20

cmake . -DCMAKE_INSTALL_PREFIX=/mysql/app/mysql -DENABLED_LOCAL_INFILE=1 -DMYSQL_DATADIR=/mysql/data/3306/data -DWITH_INNOBASE_STORAGE_ENGINE=1 -DSYSCONFDIR=/mysql/data/3306 -DMYSQL_UNIX_ADDR=/mysql/data/3306/mysql.sock -DMYSQL_TCP_PORT=3306 -DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci -DWITH_EXTRA_CHARSETS=all -DDOWNLOAD_BOOST=1 -DWITH_BOOST=/mysql/app/mysql-5.7.20/boost/boost_1_59_0


#进入编译好的mysql目录
make

make install
```

5.初始化mysql

```shell
mysqld --defaults-file=../my.cnf --initialize --user=mysql --basedir=../mysql --datadir=../data

#其中my.cnf中的log-error=/mysql/log/3306/db01-error.err需要手工创建，并授予mysql用户权限
```

6.设置开机自动启动

```shell
chkconfig --list | grep mysql
chkconfig --level 35 mysql on
```

## yum方式安装

1.mysql官方下载：https://dev.mysql.com/downloads/file/?id=484922

2.下载下来后需要连网安装

```shell
#配置yum源
rpm -ivh mysql80-community-release-el8-1.noarch.rpm

cd /etc/yum.repos.d/

mysql-community.repo
mysql-community-source.repo


yum repolist all | grep mysql

yum install mysql-community-server
```

## rpm方式安装

1.下载mysql的RPM：https://cdn.mysql.com//Downloads/MySQL-5.7/mysql-5.7.30-1.el7.x86_64.rpm-bundle.tar

2.解压

```shell
tar -xvf mysql-5.7.30-1.el7.x86_64.rpm-bundle.tar
```

3.安装server和client

```shell
#出现以下的错误；
[root@docker soft]# rpm -ivh mysql-community-{server,client,common,libs}-*
警告：mysql-community-server-5.7.30-1.el7.x86_64.rpm: 头V3 DSA/SHA1 Signature, 密钥 ID 5072e1f5: NOKEY
错误：依赖检测失败：
	mariadb-libs 被 mysql-community-libs-5.7.30-1.el7.x86_64 取代
	mariadb-libs 被 mysql-community-libs-compat-5.7.30-1.el7.x86_64 取代

#解决方法
yum remove mysql-libs

#重新安装
rpm -ivh mysql-community-{server,client,common,libs}-*
警告：mysql-community-server-5.7.30-1.el7.x86_64.rpm: 头V3 DSA/SHA1 Signature, 密钥 ID 5072e1f5: NOKEY
准备中...                          ################################# [100%]
正在升级/安装...
   1:mysql-community-common-5.7.30-1.e################################# [ 20%]
   2:mysql-community-libs-5.7.30-1.el7################################# [ 40%]
   3:mysql-community-client-5.7.30-1.e################################# [ 60%]
   4:mysql-community-server-5.7.30-1.e################################# [ 80%]
   5:mysql-community-libs-compat-5.7.3################################# [100%]
```

4.启动服务

```shell
service mysqld start

#后续操作
#1.找到初始密码
cd /var/log
more mysqld.log |grep password 
2020-05-05T05:33:36.791446Z 1 [Note] A temporary password is generated for root@localhost: uGrWh:KhE4<;

#设置密码
set password=password('~!@123abcA');
```

5.卸载

```shell
rpm -qa | grep mysql

#接着使用yum remove卸载
yum remove mysql-community*
```















