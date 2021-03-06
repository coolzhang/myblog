# Squirrel介绍
Squirrel是一个基于Redis Cluster实现的Cache解决方案，是美大上海存储团队维护的一个内部项目，目前仍处于不断完善阶段暂未开源。有几部分组成：

* **Squirrel客户端**（***RD关心部分***）：基于Jedis封装的SmartClient。
* **配置中心**：存储相关配置信息，包括：集群节点信息、Category配置以及客户端链接信息和路由策略。
* **Squirrel管理端**（***运维人员关心部分***）：提供了一个集群管理界面，支持集群创建、配置管理、监控告警。
* **Redis Cluster**：实现数据分片，高可用。

【架构图如下】  

![Squirrel Architecture](https://github.com/coolzhang/myblog/blob/master/misc/squirrel_arch.png)

## Squirrel优势

1. 支持业务Key区分  
Squirrel配置中心通过设计了一个叫做【Category】功能来支持区分来自不同业务的Key。实际存储在Redis中的Key由3部分组成：category、template、version，例如：ordercenter.c{0}d{1}\_0，其中{0}{1}..{n}这些是占位符，由开发人员提供，可以对应数据库表中的某个字段或者是别的什么可区别的，version默认是0，通过修改版本号可以批量删除对应的Key(实际上并没有删除，只是对应业务来说Key变化了查询不到了)。【Category】功能其实有点类似于数据库中表的效果，将集群中不同用途的Key区分开来。  
2. 支持灵活的路由策略  
Squirrel通过配置不同的路由策略支持读写分离，读的负载均衡。

## Squirrel局限

Squirrel客户端如果需要支持多种语言，需要依赖于不同语言的连接驱动进行多次研发，目前仅支持JAVA，对于语言统一的开发团队比较合适。

## 类似方案

由搜狐视频开源的一个Redis云管理平台[CacheCloud](https://github.com/sohutv/cachecloud)。(非常不错，很赞！)

# Codis介绍
[Codis](https://github.com/CodisLabs/codis/blob/release3.2/doc/tutorial_zh.md)是由豌豆荚开源的一款分布式Redis集群解决方案（官方Redis Cluster没有推出之前最佳解决方案）。由以下组件组成：

* **Codis Server**：基于官方Redis版本进行二次开发。
* **Codis Proxy**：Redis代理服务器，客户端通过proxy访问后端redis server。
* **Codis Dashboard**：集群管理工具，配合FE图形化界面管理集群，同时保证所有proxy状态一致。
* **Codis Admin**：支持集群管理的命令行工具。
* **Codis FE**：集群管理界面。
* **Storage**：配置中心，支持三种实现zookeeper、etcd、Fs，另外，可以通过Namespace来划分不同业务。

【架构图如下】 

![Codis Architecture](https://github.com/CodisLabs/codis/blob/release3.2/doc/pictures/architecture.png)

## Codis优势

1. 无语言驱动限制  
由于Codis是通过proxy来实现分片逻辑，故不再受语言驱动的限制，方便任何客户端驱动连接。
2. proxy无状态，支持水平扩展
3. 支持读写分离
4. 通过Namespace概念，不同集群可以按照不同的业务进行划分

## Codis劣势

1. 由于客户端到后端Redis server多了一层proxy通讯，对于性能会有一定损耗。
2. Codis server是对官方版本有修改，后期版本的维护也是使用者需要考虑的一个因素。
