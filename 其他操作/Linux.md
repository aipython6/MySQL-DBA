# Linux

## 防火墙如何设置规则



## yum源

yum源可以安装和卸载rpm包，yum源可以配置本地和网络源

常用命令

```
yum list
yum search	关键字
yum -y update
yum -y remove  包名 ---卸载
yum clean all
yum grouplist
yum groupinstall
yum groupremove
```



## RPM包

RPM安装：rpm -ivh 包名

RPM升级：rpm -Uvh 包名

RPM卸载：rmp -e 包名

RPM查询：rpm -q(-qa,-qi,-qf) 包名

软件包依赖：rpm -qR 包名

RPM包安装位置

```
/etc/	配置文件安装目录
/etc/bin/	可执行的命令安装目录
/usr/lib/	程序所使用的函数库保存位置
/usr/share/doc/	基本的软件使用手册保存位置
/usr/share/man/	帮助文件保存位置
```



## 源码包

```
源码包的默认保存位置：
/usr/local/src/
软件安装位置：
/usr/local


源码包安装过程
1、下载源码包
2、解压下载的源码包
3、进入解压目录
./configure --prefix=安装位置
(该功能是软件配置与检查定义需要的功能和选项，检查系统华景是否符合安装要求，把定义好的功能选项和检测系统环境的信息写入makefile文件，用于后续编辑)

make编译成二进制文件，当安装出错时，可以make clean让安装环境清除

make install执行安装，并拷贝到指定的目录
/usr/local/mysql/bin/mysqlstl start

```















