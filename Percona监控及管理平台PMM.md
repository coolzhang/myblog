 
**PMM（Persona Monitoring and Management Platform）是一个高效的数据库监控与管理平台，基于开源时序数据库Prometheus作为后端数据存储，通过Consul实现客户端配置管理，并依靠Grafana提供图形展示与状态报警，同时借助Orchestrator进行MySQL复制架构的可视化管理。**

## 整体架构
![PMM Architecture](https://www.percona.com/doc/percona-monitoring-and-management/_images/pmm-diagram.png)

两个角色：`PMM Client` 和 `PMM Server`

* ***PMM Client*** - 类似zabbix-agent，通过pmm-admin管理工具将被监控机器添加到PMM Server监控
列表中，通过各种exporter对主机及服务进行监控，目前PMM支持的export如下：

```
* persona-wan-agent - 慢查询分析程序；
* node_exporter - 主机指标收集程序；
* mysqld_exporter - mysql指标收集程序；
* mongodb_exporter - mongodb指标收集程序;
* proxysql_exporter -  proxysql性能指标收集程序。
```

* ***PMM Server*** - 通过Prometheus(时序数据库)进行数据的采集和存储，并调用Consul来实时同步监控主机列表，最后由Grafana来进行图形展示；另外还附带一个慢查询监控系统。

## 安装部署
###PMM Server - 通过docker image来安装部署，步骤如下：

#### Step1. Create a PMM Data Container
```
# docker create \
   -v /opt/prometheus/data \
   -v /opt/consul-data \
   -v /var/lib/mysql \
   -v /var/lib/grafana \
   --name pmm-data \
   percona/pmm-server:1.0.5 /bin/true
```
#### Step2. Create and Run the PMM Server Container
```
# docker run -d \
   -p 80:80 \
   --volumes-from pmm-data \
   --name pmm-server \
   --restart always \
   percona/pmm-server:1.0.5
```
#### Step3. Verify Installation
Component                     | URL
------------------------------|-----------------------------------
PMM landing page					| http://192.168.100.1
Query Analytics(QAN web app)	| http://192.168.100.1/qan/
Metrics Monitor(Grafana)		| http://192.168.100.1/graph/
Prometheus						| http://192.168.100.1/prometheus/
Orchestrator						| http://192.168.100.1/orchestrator

### PMM Client

####二进制包安装，如下：
```
#shell> wget https://www.percona.com/downloads/pmm-client/pmm-client-1.0.5/binary/redhat/6/x86_64/pmm-client-1.0.5-1.x86_64.rpm
#shell> rpm -ivh pmm-client-1.0.5-1.x86_64.rpm
默认安装目录：/usr/local/percona；管理工具为：/usr/sbin/pmm-admin

升级PMM Client，如下：
#shell> wget pmm-client-1.0.5
#shell> rpm -e pmm-client-1.0.4-1.x86_64
#shell> rpm -ivh pmm-client-1.0.5-1.x86_64.rpm
#shell> pmm-admin config --server PMM-SERVER-IP
#shell> pmm-admin repair [mysql | mongodb]
#shell> pmm-admin add [mysql | mongodb]
```

##日常维护命令

* 添加监控机
`pmm-admin config --server 192.168.100.1`

* mysql监控
`pmm-admin add mysql --user pmm --password pass4pmm --host 127.0.0.1 --port 3306`

* CDB监控
`pmm-admin add mysql:metrics --host 10.66.120.110 --user pmm --password pass4pmm CDB-app-mysql`
`pmm-admin add mysql:queries --host 10.66.120.110 --user pmm --password pass4pmm CDB-app-mysql`

* mongodb监控
`pmm-admin add mongodb --replset repset --nodetype mongod --uri mongodb://user:xxxx@127.0.0.1/admin`

* 查看监控状态
`pmm-admin list`

* 网络状态检查
`pmm-admin check-nework --no-emoji`

* Server端连通
`pmm-admin ping`

* 服务开关
`pmm-admin start/stop --all` 

* 清除服务
`pmm-admin remove mysql|mongo`

## 备注
MySQL服务包括：linux:metrics, mysql:metrics, mysql:queries；mongodb服务包括：linux:metrics, mongodb:metrics，分别对应不同的端口，默认如下：

服务				|端口
----------------	|--------
linux:metrics		|42000
mysql:queries		|42001
mysql:metrics		|42002
mongodb:metrics	|42003
 