***注：线上版本2.8*** 

## Redis介绍

Redis是一款开源的内存型数据结构存储，可以作为数据库、缓存以及消息服务器。具有以下特点：

1. 支持丰富的数据结构（strings、hashes、lists、sets、sorted sets、bitmaps、hyperloglogs、geospatial indexes）；
2. 支持LUA脚本、LRU内存淘汰策略、事务操作、Pub/Sub；
3. 支持数据持久化，一种是RDB镜像，另一种是AOF；
4. 支持主从复制，通过Sentinel来实现故障切换；
5. 3.0后的Redis Cluster支持数据水平扩展。

### 持久化
1. RDB持久化是将某一时间点上的数据库状态（数据库中的键值对）保存到一个RDB文件中，该文件是一个经过压缩的二进制文件；既可以手动执行SAVE命令，也可以通过配置选项（e.g: save 900 1）定期执行，SLAVEOF命令也会触发主节点执行BGSAVE命令。
2. AOF（Append Only File）持久化是通过保存所有修改数据库的写命令请求来记录服务器的数据库状态，AOF文件是以Redis的命令请求协议格式保存的纯文本格式；通过配置选项appendonly开启AOF持久化。

### 阻塞服务器的命令
1. SAVE命令会阻塞Redis服务器进程，直到RDB文件创建完毕为止，在服务器进程阻塞期间，服务器不能处理任何命令请求，而BGSAVE命令会派生出一个子进程，然后有子进程复制创建RDB文件，所以该命令不会阻塞服务器。
2. BGREWRITEAOF命令在整个AOF后台重写过程中，只有信号处理函数执行是会对服务器进程造成阻塞，在其他时候，AOF后台重写都不会阻塞父进程。

### 日常管理命令
```
127.0.0.1:6379> info [keyspace]                          # 查看实例配置及使用信息
127.0.0.1:6379> dbsize                                   # 显示当前数据库下key的个数
127.0.0.1:6379> select db_num                            # 切换到指定数据库下（默认为0）
127.0.0.1:6379> type key_name                            # 查看key的类型
127.0.0.1:6379> role                                     # 查看实例角色信息
127.0.0.1:6379> monitor                                  # 实时输出命令交互信息（线上禁止使用）
127.0.0.1:6379> shutdown                                 # 关闭实例
127.0.0.1:6379> keys mykey*                              # 线上禁止使用，推荐scan，如下：
127.0.0.1:6379> scan 0 match mykey*                      # 默认一次输出10条，第一个输出值，作为下一次scan的参数
127.0.0.1:6379> slowlog get/len/reset                    # get获得慢查询语句，len获得当前慢语句的条数，reset清空慢语句列表
127.0.0.1:6379> client kill type [normal,master,slave,pubsub]        # 根据请求类型批量kill连接
shell> redis-cli -p 6379 debug sleep 120                 # 模拟master无响应120秒，导致sentinel切换
shell> redis-cli -n 2 --scan | xargs redis-cli -n 2 del  # 清理某个DB下的大量数据时，线上不要使用flushdb，否则会阻塞业务
shell> redis-cli -n 1 keys "pay_no_20*" | xargs redis-cli -n 1 del   # 清理满足需求的key
```

## Redis Sentinel

Redis主从切换使用Redis 2.8/3.0中自带的sentinel实现自动切换，Sentinel具有如下功能：

1. Redis实例以及sentinel进程检测；
2. 提供了调用外部脚本的API接口，检查到任何异常（比如：实例宕机、主从切换、sentinel进程关闭重启）均会触发调用事先写好的脚本（比如：邮件通知、执行相关操作）；
3. 当达到quorum值，同时必须有大多数sentinel响应之后，才会完成自动主从切换；
切换发生后，sentinel进程会自动更新自己的配置文件（比如：sentinel.conf）

## Redis Cluster

### Redis Cluster可能会遇到问题

1. 对于开发来说，目前支持cluster的语言驱动还不完善，JAVA推荐Jedis，PHP推荐Predis。
Jedis的一些不足如下（网上收集到的，Predis没有太多信息）：
JedisCluster 的info()等单机函数无法调用,返回(No way to dispatch this command to Redis Cluster)错误；
JedisCluster 没有针对byte[]的API，需要自己扩展。
2. 对于运维来说，key是随机被存储到cluster中的节点上，没有命令可以查看节点上都分布了哪些key，一旦有节点故障无法预估对于业务的影响程度。
3. 另外，cluster中的slave对外不能提供任何请求。

### 驱动使用方式

#### 1.Predis连接代码，如下：
```
<?php  
// WARNING: right now, support for redis-cluster is experimental,
// unoptimized and in its very early stages of development. To
// play with it you must have a correct setup of redis-cluster
// running (oh hai mr. obvious) and Predis v0.7-dev fetched from
// the redis_cluster branch of the Git repository.
require 'autoload.php';
$servers = array(
    'tcp://10.1.1.56:7000',
    'tcp://10.1.1.43:7001',
    'tcp://10.1.1.57:7002',
);
// Developers can specify which kind of cluster strategy the
// client should use with the recently added 'cluster' option
// and the following values:
//   - predis : good old client-side sharding (default)
//   - redis  : redis-cluster
$client = new Predis\Client($servers, array('cluster' => 'redis'));
$client->set("foo", "bar");
$foo = $client->get("foo");
```

#### 2.Jedis连接代码，如下：
```
Set<HostAndPort> jedisClusterNodes = new HashSet<HostAndPort>();
//Jedis Cluster will attempt to discover cluster nodes automatically
jedisClusterNodes.add(new HostAndPort("10.104.49.56", 7000));
JedisCluster jc = new JedisCluster(jedisClusterNodes);
jc.set("foo", "bar");
String value = jc.get("foo");
```

### 参考连接

* Redis cluster tutorial： <http://redis.io/topics/cluster-tutorial#redis-cluster-tutorial>
* redis-cluster研究和使用：<http://hot66hot.iteye.com/blog/2050676>
* redis cluster使用经验：<http://yangzhe1991.org/blog/2015/04/redis-cluster/?replytocom=25558>
* 唯品会Redis Cluster大规模实践：<http://www.jianshu.com/p/ee2aa7fe341b>
* 360在使用redis cluster之前你需要知道这些事：<https://sanwen8.cn/p/3bc2SBx.html>
