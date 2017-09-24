## Zabbix小知识

### 监控项(`Item`)的常用类型有如下几种

* **zabbix agent** - 被动类型，由server端发起获取数据请求，agent端才响应收集数据；
* **zabbix agent (active)** - 主动类型，agent端收到server端模板定义的Item项后，定期将收集的数据主动发给server端；
* **zabbix trapper** - 将收集到的数据以一定的格式(hostname key value)，通过zabbix_sender命令发送给server端，采集数据的操作可以在server端也可以在agent端；
* **external check** - 通过在zabbix_server.conf中定义ExternalScripts选项，将shell脚本或是二进制包放在该目录下，server端会定期执行脚本。

### 数据收集周期

均通过对应模板中的【`Update interval` (in sec)】字段来定义数据收集频率，不同模板触发收集的方式各有不同，差别如下：

* **Fromdual Templates** - MySQL监控模板，通过子模板【Template_FromDual.MySQL.mpm】中的Item【MPM Agent is alive】（类型为zabbix agent，其它监控项类型为zabbix trapper），由server端定期触发收集数据，在agent端需要定义UserParameter，架构图如下：

![MPM Architecture](http://www.fromdual.com/sites/default/files/mpm_inst_guide00.png)

* **Mikoomi Templates** - MongoDB监控模板，通过模板【Mikoomi MongoDB Template】中的Item【Miscellaneous: Data Collector】（类型为External check，其它监控项类型为zabbix trapper），由server端定期触发收集数据；

* **Template Redis 2** - Redis监控模板，通过操作系统的crontab每分钟调用zbx_redis_stats.py脚本收集数据（类型为zabbix trapper）***注：线上并未采用此模板，而是自己定义模板，采用shell脚本获取数据。***

### UserParameter定义

`UserParameter`中定义的Key的类型，必须是zabbix agent或者zabbix agent (active)
https://www.zabbix.com/documentation/2.2/manual/config/items/userparameters

## Zabbix主机名定义规则

#### 1.监控主机命名

Zabbix agent主机名（`Hostname`）目前线上定义规则：`业务名称`-`数据库产品及角色`-`主机IP`-`端口`，例如：

Hostname                            |注释
------------------------------------|-----------
wxmovie-mysqlmaster-10.1.1.1-3306   |MySQL主
wxmovie-redisslave-10.1.1.2-6379    |Redis从

新考虑的命名规则：`数据库产品`-`服务角色`-`主机IP`，例如：

Hostname           |注释
-------------------|-----------
my-m-10.1.1.1      |MySQL主
rd-s-10.1.1.2      |Redis从
mg-m-10.1.1.3      |MongoDB主

#### 2.监控策略

**主机发现：** 通过Action中的`Auto registration`方式，被监控端主动到server端注册发现；通过定义的主机匹配规则，来关联相应的模板；

**服务发现：** 采用Zabbix 2.0后支持的`LLD`（low-level discovery）技术，通过自定义的自动发现模板，会根据端口自动关联监控项、报警、趋势图。

#### 3.命名的设计缘由

之前的命名规则为：业务-角色-IP-端口，在单机单实例的场景下非常的合适；但是，未来可能在一个很长时间我们数据库资源的使用策略会是减少机器数量，充分利用现有资源，将会是单机多实例的场景。对应监控来说，如果还沿用之前的传统监控模板，显然不够灵活，所以我们利用最新的LLD技术，重新定义了一套数据库服务自动发现模板。在新的监控思路下，主机名不在包含更多定义，而单纯的就是一个服务提供者（MySQL、NoSQL、MQ），其实单纯的IP（Hostname唯一性）就够了，但又为关联特定的模板进而引入数据库产品名称和服务角色。而对于监控主机下具体业务的识别，可以通过模板中关联的端口清楚的知道被哪个业务使用。关于端口划分根据数据库产品与业务来划分区间。

## MongoDB监控

### 1.监控模板

[**Mikoomi MongoDB Plugin**](https://code.google.com/p/mikoomi/wiki/03)

### 2.脚本依赖安装包
shell> yum install -y zabbix-agent zabbix-sender php php-devel php-pear

* 安装PHP的MongoDB驱动

**源码安装**

```
shell> wget https://github.com/mongodb/mongo-php-driver/archive/master.zip
shell> unzip mongo-php-driver-master.zip
shell>cd mongo-php-driver-master
shell> phpize && ./configure && make all && make install
shell> vim /etc/php.ini
---------------------------
;;;;;;;;;;;;;;;;;;;;;;
; Dynamic Extensions ;
;;;;;;;;;;;;;;;;;;;;;;
extension=mongo.so
---------------------------
```

**通过PECL安装**

```
shell> pecl install mongo
```

**查看mongo扩展版本**

```
shell> php --ri mongo |grep Version
```

### 3.脚本部署目录(Zabbix Server端)

```
/etc/zabbix/
|-- externalscripts
     |-- mikoomi-mongodb-plugin.php
     |-- mikoomi-mongodb-plugin.sh
```

### 4.Zabbix Server端配置

```
shell> vim /etc/zabbix/zabbix_server.conf
-----------------------
# External checks
ExternalScripts=/etc/zabbix/externalscripts
-----------------------
```

### 5.Zabbix UI配置

```
Configuration --> Templates --> Import --> MongoDB_Plugin_template_export.xml
Configuration --> Templates(Mikoomi Templates) --> Items --> Miscellaneous: Data Collector  --> Key: mikoomi-mongodb-plugin.sh[--,-h,{$MONGODB_SERVER},-p,{$MONGODB_PORT},-z,{$MONGODB_ZABBIX_NAME},-u,{$MONGODB_USER},-x,{$MONGODB_PASSWORD}]
Configuration --> Host groups --> Create host group --> ”wxmovie_Mongo“
Configuration --> Hosts --> ommongo-master-10.1.1.2-27017 --> Macros
 
Macro                  |Value
-----------------------|----------------------
{$MONGODB_PASSWORD}    |KrOWpKXwNd
{$MONGODB_PORT}        |27017
{$MONGODB_SERVER}      |10.1.1.2
{$MONGODB_USER}        |monitor
{$MONGODB_ZABBIX_NAME} |ommongo-master-10.1.1.2-27017

注：创建监控用户，如下：
shell> mongo admin -u root -p xxxx
rs0:PRIMARY> db.createUser(
          {
             user: "monitor",
             pwd: "xxxx",
             roles: [ { role: "clusterMonitor", db: "admin"} ]
          }
      )
```

### 6.问题排查

* Zabbix Server日志：  
`/var/log/zabbix/zabbix_server.log`

* Mikoomi数据/日志：
`/tmp/mikoomi-mongodb-plugin.php_ommongo-master-10.143.68.208-27017.data`
`/tmp/mikoomi-mongodb-plugin.php_ommongo-master-10.143.68.208-27017.log`

## RabbitMQ监控

* 方法一：
<https://github.com/jasonmcintosh/rabbitmq-zabbix>

* 方法二：
<http://blog.thomasvandoren.com/monitoring-rabbitmq-queues-with-zabbix.html>

## MySQL与Redis监控

* `MySQL` - <https://github.com/coolzhang/saltbox/tree/master/mysql>
* `Redis` - <https://github.com/coolzhang/saltbox/tree/master/redis>

注：关于MySQL与Redis的监控见SaltStack自动化安装步骤中的`monitor.sls`。
