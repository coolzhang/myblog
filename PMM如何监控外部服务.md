## PMM添加外部监控服务  

PMM默认支持主机监控、MySQL相关监控（包括：MySQL[MyISAM,InnoDB,MyRocks,TokuDB]、MariaDB、PXC、ProxySQL）、MongoDB相关监控（包括：[MMAPv1,WiredTiger,InMemory,RocksDB]），为了可以方便的接入更多服务，PMM提供了外部监控服务接口，通过external:metrics和external:instances两个命令实现对非内部服务的监控。  

### Redis监控  

* 编写收集Redis监控指标的`Exporter`，
  采用第三方`redis_exporter` <https://github.com/oliver006/redis_exporter>。
* 导入对应`Dashboard`模板，redis\_exporter自带了模板，在`contrib`目录下，下面有两个模板带别名和不带别名，我们选择带别名的`grafana_prometheus_redis_dashboard_alias.json`，通过定义别名来描述该实例属于哪个业务。
* 通过`pmm-admin`将服务添加到到Prometheus的配置文件(`/etc/prometheus.xml`)中，具体配置如下：  

> 启动exporter监控服务

```
shell> /usr/local/percona/pmm-client/redis_exporter -web.listen-address 10.1.1.6:9121 -redis.addr redis://10.1.1.6:6379 -redis.alias cmug-rediscluster01  
```  

> 添加外部监控服务  
 
```
shell> pmm-admin add external:metrics redis --interval 1s --timeout 1s --path /metrics --scheme http  
shell> pmm-admin add external:instances redis 10.1.1.6:9121
```  

**注：**`external:metrics`和`external:instances`分别对应prometheus.xml配置文件中的`job_name`和`targets`标签，`9121`是PMM默认外部监控服务使用的通信端口可自定义。

### Kafka与Zookeeper监控  

#### 向Prometheus中添加kafka、zookeeper外部服务  
```
 # kafka服务
shell> pmm-admin add external:metrics kafka --interval 1s --timeout 1s --path /metrics --scheme http
shell> pmm-admin add external:instances kafka 10.1.1.6:9999
 # zookeeper服务
shell> pmm-admin add external:metrics zookeeper --interval 1s --timeout 1s --path /metrics --scheme http
shell> pmm-admin add external:instances zookeeper 10.1.1.6:9998
```

#### 配置kafka、zookeeper对应的JMX Exporter  

Kafka与Zookeeper监控通过JMX特性获取内存metrics信息，Prometheus提供了现成的jmx_exporter，另外还需配置对应的YAML配置文件，提供抓取的metrics信息。具体配置如下：  

> 下载jmx_exporter

```
shell> wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.10/jmx_prometheus_javaagent-0.10.jar
```

> 定制jmx\_prometheus\_javaagent所需的YAML配置文件  

```
 # prometheus提供了一些服务的YAML文件例子
<https://github.com/prometheus/jmx_exporter/tree/master/example_configs>  
```

> 开启服务的JMX功能  

```
 # kafka启动开启JMX_PORT
shell> KAFKA_OPTS="$KAFKA_OPTS -javaagent:/usr/local/kafka/jmx_exporter/jmx_prometheus_javaagent-0.6.jar=9999:/usr/local/kafka/jmx_exporter/kafka-0-8-2.yml" /usr/local/kafka/bin/kafka-server-start.sh -daemon /usr/local/kafka/config/server.properties

 # zookeeper启动开启JMX_PORT
shell> vim /usr/local/zookeeper/bin/zkServer.sh  
	...
	[[JMX_OPTS="-javaagent:/usr/local/zookeeper/jmx_exporter/jmx_prometheus_javaagent-0.10.jar=9998:/usr/local/zookeeper/jmx_exporter/zookeeper.yml"]]
	case $1 in
	start)
	    echo  -n "Starting zookeeper ... "
	...
	    nohup "$JAVA" "-Dzookeeper.log.dir=${ZOO_LOG_DIR}" "-Dzookeeper.root.logger=${ZOO_LOG4J_PROP}" \
	    -cp "$CLASSPATH" $JVMFLAGS [[$JMX_OPTS]]  $ZOOMAIN "$ZOOCFG" > "$_ZOO_DAEMON_OUT" 2>&1 < /dev/null &
	...

shell> /usr/local/zookeeper/bin/zkServer.sh start
```
**注：**[[ ]]之间的部分为向原生`zkServer.sh`脚本新增内容。  

### 相关连接  

1. [How to Monitor Zookeeper](https://blog.serverdensity.com/how-to-monitor-zookeeper/)
2. [ZooKeeper JMX](https://zookeeper.apache.org/doc/r3.3.2/zookeeperJMX.html)
3. [Monitoring Kafka Streams Metrics via JMX](https://www.madewithtea.com/monitoring-kafka-streams-metrics-via-jmx.html#resources)
4. [Monitoring Kafka with Prometheus](https://www.robustperception.io/monitoring-kafka-with-prometheus/)
5. [Monitoring Apache Kafka with Prometheus](https://blog.rntech.co.uk/2016/10/20/monitoring-apache-kafka-with-prometheus/)
6. [jmx_exporter](https://github.com/prometheus/jmx_exporter)
7. [Prometheus and JMX](http://www.whiteboardcoder.com/2017/04/prometheus-and-jmx.html)