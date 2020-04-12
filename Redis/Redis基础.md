# Redis基础

## Redis特点

- 速度快
- 基于key-value的数据结构服务器
- 简单稳定
- 持久化（RDB和AOF）
- 主从复制
- 高可用和分布式

## Redis使用场景

- 缓存
- 排行榜系统
- 计数器应用
- 社交网络
- 消息队列系统
- 



## Redis的安装

### Redis的启动

```shell
redis-server /mysql/data/redis/conf/redis.conf
```



### Redis关闭

```
redis-cli shutdown nosave|save
```



## Redis命令

### 全局命令

```
查看所有键
keys *

键总数
dbsize

检查键是否存在
exists key

删除键
del key

键过期
expire a 10

检查键还有多长时间过期
ttl a

键的数据类型

```























































