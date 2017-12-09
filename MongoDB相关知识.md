## MongoDB介绍

MongoDB是一款开源的文档型数据库，属于NoSQL范畴，但其又具有很多类关系型数据库的特性。其特点如下：

1. 灵活的文档组织形式，类似JSON对象，文档是由一系列field:value对组成（e.g.: {field1:value1, field2:value2, ...}），value可以是字符串、数字、数组、文档，这与程序语言中支持的数据类型很相似；
2. 自由的schema定义，有利于业务功能快速迭代更新；
3. 支持丰富的类关系型的读写语句，同时还支持数据聚合操作，以及全文索引和地理位置索引；
4. 高可用副本集功能支持自动故障切换与多节点数据冗余；
5. 自带Sharded Cluster方案支持水平扩展，实现了自动化的动态扩容和负载均衡；
6. 跟MySQL一样MongoDB也支持多引擎（e.g.: MMAPv1、WiredTiger、In-Memory、RocksDB）。

## MongoDB副本集安装注意点

1. 创建启动用户mongo
2. 禁用内核参数transparent_hugepage
3. 关闭NUMA
4. 生成认证文件，权限400（openssl rand -base64 756 > key4rs）
5. 启动选项(从3.2开始默认引擎为WiredTiger)

```
 --storageEngine wiredTiger 
 --wiredTigerCacheSizeGB x.x 
 --wiredTigerDirectoryForIndexes 
 --dbpath /data/mongo27017 
 --logpath /data/mongo27017/log/mongod.log 
 --replSet rs0 
 --keyFile /data/mongo27017/key4rs 
 --journal --fork
```

## 配置副本集

初始化副本集，执行初始化的节点将被提升为Primary(该节点可以存在已创建的用户)，另外其他节点不能存在任何数据(包括新建的用户，即无需创建任何用户，最终会同步Primary上的已有用户，Arbiter节点除外，需要手动创建用户)，例如：

```
shell> mongo 127.0.0.1:27017/admin -uroot -pxx  # 连接任意一个节点
> rs.initiate(
    {
           _id: "rs0",
          members: [
                  { _id: 0, host : "10.1.1.216:27017" },
                  { _id: 1, host : "10.1.1.20:27017" }
          ]
    }
)
rs0:PRIMARY > rs.addArb("10.1.1.250:27017")     # 添加仲裁(Arbiter)节点，不存储数据只参加选举投票
rs0:PRIMARY > rs.status() | rs.conf             # 查看集群状态或配置
```
***注：线上3.0版本副本集架构***
![ReplSet](https://docs.mongodb.com/manual/_images/replica-set-primary-with-secondary-and-arbiter.bakedsvg.svg)

## 副本集知识点汇总  

1. 各节点每隔2秒彼此ping对方，如果heartbeat探测超过10秒没有响应，节点会被判定为不可用，进而会发生选举操作；故障切换时间在1分钟以内完成，选举时间大概10-30秒。  
2. [选举协议](https://docs.mongodb.com/manual/reference/replica-set-protocol-versions/)从3.2以后使用pv1，之前采用pv0，pv1的选举效率更高完成切换时间更短。  
3. 默认读写都在Primary节点，可以通过配置readPreference来调整读操作访问的节点，提供了[5种选项](https://docs.mongodb.com/manual/core/read-preference/#read-preference-modes)。
4. 除了Primary|Secondary|Abiter这3种角色外，还有几类特殊用途的节点，如：Priority为0的节点，隐藏节点，延时节点。

## 常用命令

### 1. 查看数据对象

```
shell> show databases
shell> use test
shell> show tables
```
***注：常用SQL语句对应表：*** <https://docs.mongodb.com/v2.4/reference/sql-comparison/>

### 2. 创建用户

```
repset:PRIMARY> use test      # 需要先切换到需要创建用户的指定库下
repset:PRIMARY> show users    # 查看当前库下有哪些用户
repset:PRIMARY> db.addUser(
                  { user : "test", 
                    pwd : "12345", 
                    roles : [{role: "read", db: "test"}] 
                  }
               )              # 创建用户，并赋权限，e.g: "read" - 只读用户, "readwrite" - 读写用户, "root" - 超级权限用户,
                                                      "backup" - 备份用户, "restore" - 恢复用户, "clusterMonitor" - 监控用户
repset:PRIMARY> db.removeUser("test")                           # 删除用户
repset:PRIMARY> db.changeUserPassword("username","password")    # 修改密码
```

### 3. 副本集信息查询

```
repset:PRIMARY> db.isMaster()    # 展示副本集信息，以及当前实例角色
repset:PRIMARY> rs.status()      # 返回副本集中所有节点当前状态
repset:PRIMARY> rs.slaveOk()     # 默认从库不允许查询，该命令可以开启从库查询功能
```

### 4. 创建索引

```
repset:PRIMARY> db.xx.getIndexes()                   # 查看索引
repset:PRIMARY> db.xx.ensureIndex( { col: 1 } )      # 单列索引
repset:PRIMARY> db.xx.ensureIndex( { col1: 1, col2: 1 } )          # 多列索引
repset:PRIMARY> db.xx.ensureIndex( { col: 1 }, { unique: true } )  # 唯一索引
repset:PRIMARY> db.xx.dropIndex( { col: 1 } )        # 删除指定索引
repset:PRIMARY> db.xx.dropIndexes()                  # 删除所有索引
```

### 5. 其他管理命令

```
repset:PRIMARY> db.runCommand( {  logRotate: 1 } )   # 生成新日志
repset:PRIMARY> db.currentOp()                       # 类似show processlist
repset:PRIMARY> db.killOp(opid)                      # 类似kill thdid
repset:PRIMARY> db.runCommand( { serverStatus:1 } )  # 类似show global status
```

### 6. 第三方工具

* mtools - <https://github.com/rueckstiess/mtools>
* vc-mongo-sniffer - <https://www.vividcortex.com/resources/network-analyzer-for-mongodb>
* ognom-toolkit - <https://github.com/fipar/ognom-toolkit>  
* tuned-percona-mongodb - <https://github.com/Percona-Lab/tuned-percona-mongodb>  

## MongoDB升降级操作

### 1. 传统方式滚动式升级(2.4->2.6->3.0->3.2)

#### 1.1. ReplSet从2.4升级到2.6
方法：`逐一升级从库，最后升级主库，通过将2.4的mongod二进制文件替换为2.6即可完成升级。`

***步骤如下：***

```
1> 从库升级
shell> cd /data/soft; tar -xf mongodb-linux-x86_64-2.6.11.tar; mv mongodb-linux-x86_64-2.6.11 mongo26
shell> mongo26/bin/mongo 127.0.0.1/admin -u xx -p xx
repset:SECONDARY> db.upgradeCheckAllDBs()   # 通过2.6客户端连接，执行兼容检查命令
repset:SECONDARY> db.shutdownServer()
shell> mv mongo mongo24; mv mongo26 mongo
shell> numactl --interleave=all  /data/soft/mongo/bin/mongod --dbpath /data/mongo27017 --maxConns=10000 --fork --logpath=/data/log/mongodb.log --replSet repset --nojournal  --keyFile /data/mongo27017/r0 --syncdelay=300
shell> mongo 127.0.0.1/admin -u xx -p xx
repset:SECONDARY> rs.status()               # 检查该从库复制状态
2> 主库升级
repset:PRIMARY> rs.stepDown(300)            # 先将主库降级为从库，300表示等待300s，防止再被选为主库，后续的操作与以上从库相同
```

#### 1.2. ReplSet从2.6降级回2.4

方法：`将2.6版本的mongod文件替换回2.4版本即可完成降级。`

### 2. 通过mongosync同步数据跨版本升级

***步骤如下：***

```
1> 搭建新版本副本集集群，新实例已WT引擎启动；
shell> /data/soft/mongodb/bin/mongod --storageEngine wiredTiger \
--dbpath /data/mongo27017 --logpath /data/mongo27017/log/mongod.log \ 
--replSet rs0 --keyFile /data/mongo27017/key4rs --nojournal --fork

2> 通过mongosync将数据从原低版本集群复制到新集群；
shell> mongosync \
--src_srv 10.1.1.1:27017 --src_user xx --src_passwd xx --src_auth_db admin \
--dst_srv 10.1.1.6:27017 --dst_user root --dst_passwd xx --dst_auth_db admin \ 
--bg_num 10 --oplog 2>/data/mongosync.log

3> 程序端切换新集群主机IP
```

### 3. 升级注意事项(程序端驱动以及查询语句的兼容性)

升级MongoDB的同时，也需要升级对应的程序驱动，对应驱动兼容信息见此链接(<https://docs.mongodb.com/ecosystem/drivers/>)。

***案例如下：***

PHP的mongo.so扩展，通过php --ri mongo |grep Version检查当前版本。
升级时，PHP遇到的错误，如下：

```
2016-02-24 15:21:01 [-][-][-][error][MongoCursorException] exception 
'MongoCursorException' with message '10.104.68.250:27017: Can't canonicalize 
query: BadValue $in needs an array' in /data/www/CronTaskNew/helpers/Mongo/
Collection.php:54
```

当时线上的mongo.so版本为1.4.5，需要进行升级。

注：URI的连接配置中至少提供2个节点，否则会将连接节点视为standalone处理，不具有高可用的特性。  

## 关于事务  

**原子性** - 单个文档的写操作是原子操作，如果单个操作去变更多个文档，只能保证对当前文档的原子操作，无法保证对全部更新文档的原子操作。  

**隔离级别** - MongoDB默认为Read Uncommitted，可以通过$isolated操作符对collections加锁来控制文档的并发访问冲突。3.4开始支持Real Time Order的特性，通过将读操作readConcern配置为linearizable，写操作writeConcern配置为majority(3.4新增选项[writeConcernMajorityJournalDefault](https://docs.mongodb.com/manual/reference/replica-configuration/#rsconf.writeConcernMajorityJournalDefault)，默认为true)，可以实现对同一个文档并发读写操作时保证请求的串行化。[WiredTiger](http://source.wiredtiger.com/2.4.1/transactions.html#transaction_isolation)默认为Read Committed。
