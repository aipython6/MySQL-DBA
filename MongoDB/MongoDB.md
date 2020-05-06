# MongoDB

ln -s /mongodb/apps/mongodb-linux-x86_64-rhel70-4.0.2 mongodb /mongodb/apps/mongodb

logpath=/mongodb/logs/mongodb.log
port=27017
fork=true
#auth=true
noauth=true
#verbose=true
#vvvv=true
journal = true
maxConns=500
logappend=true
directoryperdb=true
#pidfilepath=/var/run/mongo.pid
#cpu=true
#nohttpinterface=false
#notablescan=false
#profile=0
#slowms=200
#quiet=true
#syncdelay=60
bind_ip=127.0.0.1,192.168.1.6

------------------------------
启动命令：
mongod --config /mongodb/apps/mongodb/bin/mongodb.conf

#杀mongod进程命令
kill -2 `ps -ef|grep mongod|grep -v grep|awk '{print $2}'`

mongod --port 27017 --dbpath /mongodb/data/ --logpath /mongodb/logs/mongodb.log --bind_ip 127.0.0.1,192.168.1.6
--pidfilepath /mongodb/apps/mongo.pid

分别配置主节点、从节点和冲裁节点
dbpath=/mongodb/data
logpath=/mongodb/logs/mongodb.log
port=27017
fork=true
#auth=true
#noauth=true
#verbose=true
#vvvv=true
maxConns=500
logappend=true
journal = true
pidfilepath=/mongodb/apps/mongo.pid
#cpu=true
directoryperdb=true
#nohttpinterface=false
#notablescan=false
#profile=0
#slowms=200
#quiet=true
#syncdelay=60
replSet=itpux1
bind_ip=127.0.0.1,192.168.1.6


dbpath=/mongodb/data
logpath=/mongodb/logs/mongodb.log
port=27017
fork=true
#auth=true
#noauth=true
#verbose=true
#vvvv=true
maxConns=500
logappend=true
pidfilepath=/mongodb/apps/mongo.pid
#cpu=true
journal = true
directoryperdb=true
#nohttpinterface=false
#notablescan=false
#profile=0
#slowms=200
#quiet=true
#syncdelay=60
replSet=itpux1
bind_ip=127.0.0.1,192.168.1.7

dbpath=/mongodb/data
logpath=/mongodb/logs/mongodb.log
port=27017
fork=true
#auth=true
#noauth=true
#verbose=true
#vvvv=true
journal = true
maxConns=500
logappend=true
pidfilepath=/mongodb/apps/mongo.pid
#cpu=true
directoryperdb=true
#nohttpinterface=false
#notablescan=false
#profile=0
#slowms=200
#quiet=true
#syncdelay=60
replSet=itpux1
bind_ip=127.0.0.1,192.168.1.8

如果启动时报如下的错误：
MongoDB Requested option conflicts with current storage engine option for directoryPerDB

那么就需要去数据目录下删除storage.bson，重新启动


主节点上执行：
rs.initiate({_id:'db01',members:[{_id:1,host:'192.168.1.6:27017'}]})
输出信息；
> rs.initiate({_id:'db01',members:[{_id:1,host:'192.168.1.6:27017'}]});
> {
> "ok" : 1,
> "operationTime" : Timestamp(1588495204, 1),
> "$clusterTime" : {
> 	"clusterTime" : Timestamp(1588495204, 1),
> 	"signature" : {
> 		"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
> 		"keyId" : NumberLong(0)
> 	}
> }
> }


db01:OTHER> rs.conf()
{
	"_id" : "db01",
	"version" : 1,
	"protocolVersion" : NumberLong(1),
	"writeConcernMajorityJournalDefault" : true,
	"members" : [
		{
			"_id" : 1,
			"host" : "192.168.1.6:27017",
			"arbiterOnly" : false,
			"buildIndexes" : true,
			"hidden" : false,
			"priority" : 1,
			"tags" : {
				
			},
			"slaveDelay" : NumberLong(0),
			"votes" : 1
		}
	],
	"settings" : {
		"chainingAllowed" : true,
		"heartbeatIntervalMillis" : 2000,
		"heartbeatTimeoutSecs" : 10,
		"electionTimeoutMillis" : 10000,
		"catchUpTimeoutMillis" : -1,
		"catchUpTakeoverDelayMillis" : 30000,
		"getLastErrorModes" : {
			
		},
		"getLastErrorDefaults" : {
			"w" : 1,
			"wtimeout" : 0
		},
		"replicaSetId" : ObjectId("5eae836410119e32eed581f1")
	}
}

在主节点上添加从节点：
rs.add("192.168.1.7:27017")
db01:PRIMARY> rs.add("192.168.1.7:27017")
{
	"ok" : 1,
	"operationTime" : Timestamp(1588495739, 1),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1588495739, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	}
}

在主节点上添加仲裁节点：
rs.addArb("192.168.1.8:27017")
db01:PRIMARY> rs.addArb("192.168.1.8:27017")
{
	"ok" : 1,
	"operationTime" : Timestamp(1588495757, 2),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1588495757, 2),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	}
}

rs.conf()

检查local库下面的oplog.rs集合进行数据同步
db01:PRIMARY> use local
switched to db local
db01:PRIMARY> show collections
oplog.rs
replset.election
replset.minvalid
replset.oplogTruncateAfterPoint
startup_log


db01:SECONDARY> show collections
oplog.rs
replset.minvalid
replset.oplogTruncateAfterPoint
startup_log


> use local
> switched to db local
> show collections
> replset.minvalid
> replset.oplogTruncateAfterPoint
> startup_log


use db01;
db.test1.insert({"itpux01":"fgedu01"});
for(var i =0; i <11; i ++){db.test1.insert({userName:'itpux'+i,age:i})}

如果出现：
2020-05-03T18:00:09.146+0800 I REPL_HB  [replexec-5] Error in heartbeat (requestId: 207) to 192.168.1.8:27017, response status: HostUnreachable: Error connecting to 192.168.1.8:27017 :: caused by :: No route to host
2020-05-03T18:00:09.146+0800 I ASIO     [Replication] Connecting to 192.168.1.8:27017
说明没有关闭防火墙（关闭命令：systemctl stop firewalld）

当在从节点上报如下的错误时：
db01:ARBITER> show dbs;
2020-05-03T18:17:34.521+0800 E QUERY    [js] Error: listDatabases failed:{
	"ok" : 0,
	"errmsg" : "not master and slaveOk=false",
	"code" : 13435,
	"codeName" : "NotMasterNoSlaveOk"
} :
_getErrorWithCode@src/mongo/shell/utils.js:25:13
Mongo.prototype.getDBs@src/mongo/shell/mongo.js:67:1
shellHelper.show@src/mongo/shell/utils.js:876:19
shellHelper@src/mongo/shell/utils.js:766:15
@(shellhelp2):1:1

执行：rs.slaveOk()实现同步

模拟从节点和主节点宕机，分别插入输入，再重新启动宕机的服务器，如果正常，将会发现数据可以同步
（注意：如果服务器的state状态码为8，表示服务已经不可用）

对于主节点来说，如果它宕机后再重新启动，那么启动后它不再是主节点