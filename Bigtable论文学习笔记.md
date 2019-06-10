# Bigtable论文学习笔记

> **Bigtable**是基于Google内部多个组件开发的一款分布式列式存储；数据模型使用离散、多维度有序map；既能满足高带宽的批量任务，也可以提供低延时用户请求。依赖组件如下：  

>>**Chubby** - 提供分布式锁服务，保证集群中只有一个master提供服务、存储数据的位置信息、发现正常tabletServers、存储schema信息(各表的CF信息)；  
>>**GFS** - 存储日志和数据文件；  
>>**SSTable** - 数据存储格式，支持持久化有序不可变map;  
>>**Borg** - 集群管理系统，负责管理调度服务做在服务器资源。

## 设计目标

1. 通用性；
2. 可扩展；
3. 高性能；
4. 高可用。 

## 架构

由3部分构成：客户端、master server、tablet servers。

### Master server

master主要负责tablet分配、tabletSevers变动、均衡tabletServers流量，以及垃圾文件回收；另外，当新建table和column families时负责处理schema变更。

### Tablet server

tabletServers主要负责管理tablets存储与分裂，提供读写请求。

>***Tablet Location***  

tablet位置信息通过3层结构来存储，第一层是root tablet，存储在Chubby上，包含了所有tablets的位置信息，第二层是metadata tablets，每个metadata tablet包含各自存储的user tablets的位置信息，第三层是user tables。

>***Tablet Assignment***  

Tablet一次只会被分配给一个tabletServer。每启动一个tableServer，会在Chubby系统指定的目录下创建一个锁文件，master通过监控该目录下的文件来跟踪tabletServers。只有当发生tablets合并或者分裂时才会导致tablets集合的变化。

>***Tablet Serving***  

Tablet写操作在提交跟新后会先记日志，然后再持久化到GFS中。提交后的更新会先存储在内存中的memtable中，之后会转存到磁盘上的SSTables中。

Tablet读操作需要结合SSTables和memtable中的内容来获得查询结果。

>***Compactions***

分为minor和major两种。

***minor compaction*** - 是指当memtable大小达到指定阈值时，会将原memtable锁定后转变为SSTable存储到GFS上，同时会新生成一个新的memtable。目的是为了减少内存消耗，缩短恢复时间。该操作期间不会影响正常用户请求。  

***major compaction*** - 每次minor都会生成SSTable，时间长了会存在很多SSTable，每次读取数据时都需要合并很多SSTable才可以得到最终数据。为了优化这个问题，Bigtable会定期在后台对一定数量的SSTables和memtable进行合并生成一个新的SSTable，这样被合并的SSTables既可被复用也可清理掉，从而释放空间。重写后的SSTable内容不再包含已经删除的条目。

### Client

客户端是由特定的libary实现，通过Chubby来获得tablet的存储位置，读写直接请求tabletServers，这就不会给master带来大量负载，master也就不会成为整个系统的瓶颈。客户端会缓存tablet位置信息以减少网络通信。

## 数据模型

Bigtable内部数据结构是一个松散的有序map，结构如下：

(row\_key,column\_key,timestamp) -> value

### Rows  

表(table)由若干记录(rows)组成，每条记录都对应一个row_key(可以是任意字符串，最大不超过64KB)。表可以按照记录范围，划分成若干各个tablets(分区)，tablet是管理操作的最小单元。

Bigtable起初只支持单条记录的事务操作，对于分布式事务后期通过非常规的技术方案可以达到同样效果。

### Column Families(CF)

CF由若干column_keys组成，通常这些columns的类型保持一致。CF需要先于数据创建好。建议在一个表中CF的个数不易过多(最多几百个)，而表中的columns数是没有限制的。

column_key的命名是由CF名和任意字符串构成，语法：family:qualitfier。访问控制和磁盘内存消耗统计是在CF层面实现的。

### Timestamps(TS)

每个Cell可以包含多个版本，Bigtable通过TS来实现多版本，TS是一个64位整数。应用端需要保证TS的唯一性，多个版本按照TS从大到小排序，即最新版本先被读到。多版本的回收机制可以通过配置保留时间以及保留个数来完成清理。

## 改进  

### Locality groups  

该特性用于将常访问的CF存在到同一个SSTable中，以提高查询效率，避免无关数据的访问。

### Compression  

支持对SSTable block进行压测，压缩采用two-pass的方案，该方案在保证高速压测速度的同时也可以提供了优于常规压缩工具的压缩比。

### Caching for read performance

为了提升查询性能，tableServers提供了两个层面的缓存。Scan Cache可以缓存应用的查询结果，Block Cache用于缓存SSTable blocks。

### Bloom filter

布隆过滤用于减少访问磁盘SSTables的次数。

### commit-log实现

每个tableServer有一个commit log，有两个写日志线程，但是每次只允许一个线程进行工作。

### 加速tablet恢复

通过两次compaction操作，使得需要迁移的tablet到目的tabletServer后无需回放日志。

### 充分利用不可变性

SSTables是不可变的。