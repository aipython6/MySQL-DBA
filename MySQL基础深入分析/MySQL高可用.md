# MySQL高可用

## MHA

**整体结构**

分为两部分：

- MHA Manager（管理节点）
- MHA Node（数据节点）

MHA Manager管理节点可以单独部署在一台独立的服务器上管理多个master-slave组成的集群（当然它也可以部署在一台slave上）。MHA Manager探测集群中的node节点，当发现master出现故障时，可以自动将具有最新数据的slave提升为新的master，并将其他所有的slave指向新的master，整个过程对应用程序是透明的。MHA node可以运行在每台MySQL服务器上（master/slave/manager），通过监控具备解析和清理logs功能的脚本来加快故障转移。

**原理**

维持MySQL复制中master的高可用性，可以修复多个slave之间的日志差异。当master出现异常时，MHA会选出一个与master最新日志的slave充当master，然后所有的slave都指向新的master。其他slave通过与备选库对比生成差异的relay log，并应用差异的relay log从新的master开始复制。

**优点**

- 故障切换过程中可以选择一个日志最新的slave充当master，可以减少数据的丢失，保持数据的一致性
- 支持binlog server，可提高binlog的传送效率，进一步减少数据丢失的风险

**缺点**

- 在数据传输时需要开启ssh协议，给系统的安全带来隐患

**MHA工具包**

- Manager管理工具

检查MHA的SSH配置：masterha_check_ssh

检查MySQL数据库的主从复制功能：masterha_check_repl

启动MHA服务：masterha_manager

检测当前MHA运行状态：masterha_check_status

检测master是否已经宕机：masterha_master_monitor

控制故障转移：masterha_master_switch

添加或删除配置的server信息：masterha_conf_host

- Node数据节点工具

保存和复制master的二进制日志：save_binary_logs

识别差异的中继日志事件并应用于其他的slave：apply_diff_relay_logs

取出不必要的ROLLBACK事件：filter_mysqlbinlogs

清除中继日志：purge_relay_logs



**搭建过程**

本次使用3台主机进行构建

master node：192.168.1.6（主库，并安装数据节点）

slave node：192.168.1.7（作为第一个从库，并安装数据节点）

slave2 manager node：192.168.1.8（作为第二个从库，并安装管理节点和数据节点）

1.整体布局：

![image-20200505151442028](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200505151442028.png)

2.在master、salve1和slave2上配置SSH免密登录

```shell
#3台机器生成密钥
ssh-keygen -t rsa

#将/root/.ssh下的id_rsa.pub文件拷贝到另外两台机器上（注意：在3台机器上都需要进行同样的操作）
ssh-copy-id -i id_rsa.pub root@db02
ssh-copy-id -i id_rsa.pub root@db03
#登录验证
ssh db02
ssh db03
```

3.使用GTID+row的方式实现主从复制

```mysql
-- 创建主从复制用户，并授予复制的权限
create user 'repuser'@'%' identified by '123456';
grant replication slave on *.* to 'repuser'@'%';
flush privileges;


-- 创建管理账号
create user 'zhangsan'@'%' identified by '123456';
grant all replication on *.* to 'zhangsan'@'%';
flush privileges;
```

4.在主库上将数据复制到slave1和slave2中

```shell
mysqldump --single-transaction -uroot -p123456 -A >all.sql
```

5.在slave1和slave2上执行同步数据的操作

```shell
mysql -uroot -p < all.sql
```

6.slave1和salve2上都执行主从复制的操作，实现对master数据的同步

```mysql
stop slave;
change master to
    master_host='192.168.1.6',
    master_port=3306,
    master_user='repuser',
    master_password='repuser123',
    master_auto_position=1;
start slave;
-- 检查是否已经实现同步
show slave status;


-- 注意：如果之前设置了GTID，那么需要在master上重新上重置binlog：
reset master;

-- slave上也要重置之前的环境
stop slave;
reset slave all;

-- 接着再执行
change master to
    master_host='192.168.1.6',
    master_port=3306,
    master_user='repuser',
    master_password='repuser123',
    master_auto_position=1;
start slave;

-- 检查
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.1.6
                  Master_User: repuser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: db01-binlog.000001
          Read_Master_Log_Pos: 154
               Relay_Log_File: db-relay.000002
                Relay_Log_Pos: 371
        Relay_Master_Log_File: db01-binlog.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
......
```

```shell
##另外，unrar的安装方法
wget http://www.rarlab.com/rar/rarlinux-x64-5.3.0.tar.gz
cd /rar
make

#解压文件
unrar e *.tar
```

7.安装MHA node工具(3台机器同时进行)

```
tar -xzvf mha4mysql-node-0.58.tar.gz 
cd mha4mysql-node-0.58/

perl Makefile.PL
make & make install

echo "export PATH=\$PATH:/usr/local/bin" >> ~/.bash_profile 
source ~/.bash_profile
```

8.在slave2（192.168.1.8）上安装MHA manager

```shell
mount /dev/cdrom /mnt
yum install -y perl-devel perl-CPAN perl-DBD-MySQL
yum install perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager perl-Time-HiRes -y


tar -zxvf Config-Tiny-2.23.tgz
cd Config-Tiny-2.23
perl Makefile.PL
make && make install


tar xvf mha4mysql-manager-0.58.tar.gz
cd mha4mysql-manager-0.58
perl Makefile.PL
make && make install

echo "export PATH=\$PATH:/usr/local/bin" >> ~/.bash_profile 
source ~/.bash_profile 

#将manger的脚本复制到/usr/local/bin/
cp /soft/mha4mysql-manager-0.58/samples/scripts/* /usr/local/bin/ */

#配置manager
mkdir -p /etc/masterha
cp /soft/mha4mysql-manager-0.58/samples/conf/app1.cnf /etc/masterha/app1.cnf


```

app1.cnf的具体配置

```
vi /etc/masterha/app1.cnf

[server default]
manager_workdir=/var/log/masterha/app1
manager_log=/var/log/masterha/app1/manager.log
master_binlog_dir=/mysql/log/3306/binlog
master_ip_failover_script= /usr/local/bin/master_ip_failover master_ip_online_change_script= /usr/local/bin/master_ip_online_change
password=root
user=123456
ping_interval=1
remote_workdir=/tmp
repl_password=repuser123
repl_user=repuser
report_script=/usr/local/bin/send_report
secondary_check_script= /usr/local/bin/masterha_secondary_check -s db01 -s itpuxdb03 -s db02
shutdown_script=""
ssh_user=root

[server1]
hostname=192.168.1.6
ssh_port=22
candidate_master=1
port=3306
master_binlog_dir=/mysql/log/3306/binlog

[server2]
hostname=192.168.1.7
ssh_port=22
port=3306
master_binlog_dir=/mysql/log/3306/binlog
check_repl_delay=0

[server3]
hostname=192.168.1.8
ssh_port=22
master_binlog_dir=/mysql/log/3306/binlog
no_master=1
port=3306
```

mysql -uroot -proot -e 'set global relay_log_purge=0'

slave1和slave2分别从192.168.1.6拷贝脚本

```shell
scp 192.168.1.6:/usr/local/bin/purge_relay_log.sh /usr/local/bin

chmod +x /usr/local/bin/purge_relay_log.sh
```

8.安装MHA工具检测SSH

```shell
yum -y install perl-Time-HiRes

#执行检测命令
/usr/local/bin/masterha_check_ssh --conf=/etc/masterha/app1.cnf

```

9.检测主从结构

```shell
masterha_check_repl --conf=/etc/masterha/app1.cnf
```

10.添加VIP

```shell
ip addr add 192.168.1.25 dev ens33
```

11.启动MHA服务（在192.168.1.8上执行）

```shell
bohup masterha_manager --conf=/etc/masterha/app1.cnf > /tmp/mha_manager.log < /dev/null 2>&1 &

#验证启动成功的命令
masterha_check_status --conf=/etc/masterha/app1.cnf
```

12.master出现故障，但是修复好之后，如果还希望该机器是master，则配置如下

```shell
masterha_master_switch --conf=/etc/masterha/app1.cnf --master_state=alive --new_master_host=192.168.1.6 --orig_master_is_now_slave
```

## keepalived+MHA+双主

该架构也需要基于主从复制进行搭建。

keepalived基于VRRP协议（虚拟冗余路由协议），为了解决静态路由单点故障引起的网络失效问题而设计。

两台互为主备的MySQL服务器上运行keepalived，master会想backup节点发送广播信号，当backup节点收不到master发送的VRRP包时，会认为master宕机，此时会根据VRRP的优先级来选出一个backup充当master，则这个master就会持有vip。

**搭建思路**

需要有两台MySQL服务器，两者之间互为主从，都可读可写（实际是只有一台服务器A负责数据的写入工作，而另外一台服务器B作为备用数据库）

机器1（masterA）:192.168.1.6

机器2（masterB）：192.168.1.7

vip：192.186.1.25

保持两台机器的防火墙已经关闭，而且server-id不能相同

注意：如果是新机器的话，可以直接搭建，但是如果是运行了一段时间才搭建，则需要mysqldump或xtrabackup进行备份，并在masterB上进行数据同步。

1.在主备库上创建主从复制账号

```mysql
#主库和从库都需要操作
create user 'bak'@'%' identified by '123456';
grant replication slave on *.* to 'bak'@'%';
flush privileges;

#masterB
change master to
    master_host='192.168.1.6',
    master_port=3306,
    master_user='bak',
    master_password='123456',
    master_auto_position=1;
start slave;

#masterA
change master to
    master_host='192.168.1.7',
    master_port=3306,
    master_user='bak',
    master_password='123456',
    master_auto_position=1;
start slave;
```

2.安装keepalived

```shell
yum install -y keepalived

#使用脚本来判断mysql服务是否宕机
cd /etc/keepalived
vim checkmysql.sh

#!/bin/bash
mysqlstr=/mysql/app/mysql/bin/mysql
host=192.168.1.6
user=root
password=123456
port=3306

#msyql服务状态正常为1，反之为0
mysql_status=1
#check mysql status
$mysqlstr -h $host -u -ppassword -Pport -e "show status:" > /dev/null 2>&1
if[ $? = 0 ];then
	echo "mysql_status=1"
	exit 0
else
	/etc/init.d/keepalived stop
fi
####完成脚本####

chmod +x checkmysql.sh
```

3.配置keepalived.conf

masterA

```
! Configuration file for keepalived global_defs {
router_id itpux-mysql-master
notification_email {
176140749@qq.com }
notification_email_from 176140749@qq.com 
smtp_server
stmp.qq.com smtp_connect_timeout 30
}

vrrp_instance v_mysql_wgpt1 {
state BACKUP
interface ens33
virtual_router_id 200
priority 100
advert_int 1
nopreempt authentication {
auth_type PASS auth_pass itpux
}
virtual_ipaddress{
192.168.1.25/24
	}
}
virtual_server 192.168.1.51 3306 {
delay_loop 2
lb_algo wrr
lb_kind DR
persistence_timeout 60
protocol TCP
real_server 192.168.1.6 3306 {
weight 3
notify_down /etc/keepalived/keepalived_stop.sh
TCP_CHECK {
connect_timeout 10
nb_get_retry 3
delay_before_retry 3
connect_port 3306 
		} 
	} 
}
```

msaterB

```
! Configuration file for keepalived
global_defs {
router_id itpux-mysql-master
notification_email {
176140749@qq.com }
notification_email_from 176140749@qq.com
smtp_server stmp.qq.com
smtp_connect_timeout 30 
}
vrrp_instance v_mysql_wgpt1 {
state BACKUP 
interface ens33 
virtual_router_id 200 
priority 90 
advert_int 1 nopreempt
authentication{
auth_type PASS
auth_pass db
}
virtual_ipaddress{
192.168.1.25/60
	}
}
virtual_server 192.168.1.7 3306{
delay_loop 2 
lb_algo wrr 
lb_kind DR persistence_timeout 60 
protocol TCP 
real_server 192.168.1.7 3306 { 
weight 3
}
notify_down /etc/keepalived/keepalived_stop.sh 
TCP_CHECK { 
connect_timeout 10 
nb_get_retry 3 
delay_before_retry 3 
connect_port 3306 
		}
	} 
}
```

```shell
echo "#!/bin/bash" >/etc/keepalived/keepalived_stop.sh 
echo "pkill keepalived" >>/etc/keepalived/keepalived_stop.sh 
chmod u+x /etc/keepalived/keepalived_stop.sh
```

4.启动keepalived

```shell
systemctl daemon-reload
systemctl start keepalived
systemctl enable keepalive
ps -ef |grep keepalived
```

















