**Slony-I是Postgresql的第三方企业级复制方案。PG 9.0之后推出原生复制方案Streaming Replication，通过实时同步WAL日志实现物理复制，最新的PG 10又推出了更加灵活的逻辑复制，实现了差异化复制。**

## Slony存在的价值
1. 最初PG内部并不支持复制功能，社区也不推荐其作为PG的核心功能，谁需要谁实现，例如：[Slony](https://wiki.postgresql.org/wiki/Slony)，[Bucardo](https://wiki.postgresql.org/wiki/Bucardo)，[Londiste](https://wiki.postgresql.org/wiki/Londiste)；（不能理解这个社区a- -///）
2. 支持跨版本之间的数据复制，原生Streaming Replication只支持相同版本之间同步数据；
3. 支持过滤复制，原生Streaming Replication只支持完整的实例复制。

注：PG 10已经完全可以替代Slony。

## Slony实现原理

### 内部概念  

命名                     |功能   
------------------------|----------------------
Cluster                 |参与一个database复制的所有PG实例构成一个集群，比如1主1备，1主2备，变量定义：cluster name = xxx  
Node                    |参与复制的每个PG实例分别代表一个节点，例如：1主1备，可定义为：主：node 1，备：node 2  
Replication Set         |复制集中可以定义database中需要进行复制的tables和sequences，同一个database复制中，可以定义多个复制集，例如：相同schema下的tables可以定义为一个复制集，或者tables和sequences分别定义为两个复制集 
Origin, Providers and Subscribers |复制有两种角色，一端定义为复制源，也就是数据生产者，另一端接收同步过来的数据，作为数据更新的订阅者；在Slony-I中只允许定义一个复制源，而且只能作为生产者(可以理解为单向复制)；Slony-I也支持级联复制，例如：A -> B -> C  
slon Daemon             |每个node上都需要启动一个slon进程，有两个作用，一是用来监听cluser中的配置变更，如：增减节点、复制集调整，另一个是监听表中的数据变更并将其同步到订阅者并完成更新操作   
slonik Configration Processor	|通过执行slonik命令来更新cluster配置信息，如：增删节点、通讯路径跟新、订阅者更改  
Slony-I Path Communications	|通过提供各节点的连接选项，例如 conninfo='dbname=xxx user=repluser host=replpass port=5432'，来允许节点间相互访问并执行必要命令  

### 基础配置  

* 分为3个步骤：

1. 启动slon进程；
2. 初始化Slony-I复制所需的配置信息；
3. 开启订阅。
* 示例如下：(注：master node上需要执行上述3个步骤，slave nodes上只需执行启动slon进程即可。)

#### master node

***Step-1***

	shell> sh slon_start.sh
	#!/bin/bash
	# start slon daemon on both master node and slave node
	#
	
	DBNAME=masterDB
	slon -p /tmp/slon-$DBNAME.pid slon_$DBNAME "dbname=$DBNAME host=localhost user=slony password=myslony port=15432" >> /tmp/slon-$DBNAME.log 2>&1 &
	
***Step-2***

	shell> sh slon_init.sh
	#!/bin/bash
	# replication settings of slon daemon on master node
	#
	
	DBNAME=masterDB
	MASTERNODE=10.1.1.2
	SLAVENODE=10.1.1.3
	REPLUSER=slony
	REPLPASS=myslony
	PGPORT=15432
	
	slonik <<EOF
	cluster name = $DBNAME;
	node 1 admin conninfo = 'dbname=$DBNAME host=$MASTERNODE user=$REPLUSER password=$REPLPASS port=$PGPORT';
	node 2 admin conninfo = 'dbname=$DBNAME host=$SLAVENODE user=$REPLUSER password=$REPLPASS port=$PGPORT';
	init cluster (id=1, comment = 'master node');
	create set (id=1, origin=1, comment='some tables for testing');
	set add table (set id=1, origin=1, id=1, fully qualified name = 'public.tbl_admin_autologin', comment='');
	set add table (set id=1, origin=1, id=2, fully qualified name = 'public.tbl_admin_groups', comment='');
	set add table (set id=1, origin=1, id=3, fully qualified name = 'public.tbl_admin_supplier', comment='');
	set add table (set id=1, origin=1, id=4, fully qualified name = 'public.tbl_admin_users', comment='');
	set add table (set id=1, origin=1, id=5, fully qualified name = 'public.tbl_app_me_adsetting', comment='');
	set add table (set id=1, origin=1, id=6, fully qualified name = 'public.tbl_app_me_banner', comment='');
	store node (id=2, comment = 'slave node', event node = 1);
	store path (server = 1, client = 2, conninfo = 'dbname=$DBNAME host=$MASTERNODE user=$REPLUSER password=$REPLPASS port=$PGPORT');
	store path (server = 2, client = 1, conninfo = 'dbname=$DBNAME host=$SLAVENODE user=$REPLUSER password=$REPLPASS port=$PGPORT');
	EOF

***Step-3***

	shell> sh slon_subscribe.sh
	#!/bin/bash
	# start replicating on master node
	#
	
	DBNAME=masterDB
	MASTERNODE=10.1.1.2
	SLAVENODE=10.1.1.3
	REPLUSER=slony
	REPLPASS=myslony
	PGPORT=15432
	
	slonik <<EOF
	cluster name = slon_$DBNAME;
	node 1 admin conninfo = 'dbname=$DBNAME host=$MASTERNODE user=$REPLUSER password=$REPLPASS port=$PGPORT';
	node 2 admin conninfo = 'dbname=$DBNAME host=$SLAVENODE user=$REPLUSER password=$REPLPASS port=$PGPORT';
	subscribe set ( id = 1, provider = 1, receiver = 2, forward = no);
	EOF

#### slave nodes
	shell> sh slon_start.sh
	#!/bin/bash
	# start slon daemon on both master node and slave node
	#
	
	DBNAME=H3GMovieChannel
	slon -p /tmp/slon-$DBNAME.pid slon_$DBNAME "dbname=$DBNAME host=localhost user=slony password=myslony port=15432" >> /tmp/slon-$DBNAME.log 2>&1 &

* 复制原理

一个基于触发的实现方案，采用发布/订阅的机制，通过触发slave nodes上的triggers来实现更新的回放。

![Slony Internals](https://github.com/coolzhang/myblog/blob/master/slony-i.jpg)

### 相关阅读
1. <https://momjian.us/main/writings/pgsql/replication.pdf>
2. <http://peter.eisentraut.org/blog/2015/03/03/the-history-of-replication-in-postgresql/>
