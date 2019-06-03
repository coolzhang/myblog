# GFS论文学习笔记

## 设计假设

1. 存储在普通服务器上；
2. 存储的文件数量很大，主要以大文件为主(100M以上，通常为GB大小)；
3. 读流量主要来自大量流式读(读取大小：hundreds of KBs, 1MB+)和少量随机读(读取大小：a few KBs)；
4. 写流量主要来自大量顺序写(只追加后期很少变更)和少量随机写；
5. 还不理解；
6. 高速并持久的带宽比延时更重要。

## 架构

两部分构成：一台单节点master和多台chunkservers构成。

### Master

master用来存储元数据信息，包括：命名空间(namespaces)、访问控制信息、文件与chunk的映射关系、chunk的位置；还有用于管理整个系统内部操作，包括：chunk租期管理、无用chunk回收、chunk迁移。另外，master通过heartbeat定期与chunkservers进行通信，除了发起管理操作外，还用于获得当前所有chunkservers的状态信息。

元数据主要分为三类，前两类持久化存储在本地，包括：文件与chunk的命令空间、文件与chunk的映射，第三类是非持久化，而是每次master启动后或者有新的chunkserver变动，通过与chunkservers通信后来获得所有chunk的位置信息。

operation log是用来记录元数据变更的日志，master可以通过该日志进行意外恢复；为了减少master启动时的恢复时间，operation log会定期进行checkpoint操作，为了不影响正常的请求，checkpoint由独立的线程来完成。            

### Chunkservers

chunkservers用来存储具体数据文件，GFS的客户端直接与chunkservers通信操作数据，从master仅仅是用于获得操作文件所需的元数据信息。chunkservers与客户端均不会缓存文件的数据内容，这些内容其实已经被本地文件系统缓存了。

文件存储时会被分成若干固定大小的chunk存储于chunkserver的本地磁盘，每个chunk都对应一个唯一64位句柄来标识。chunk是数据迁移、复制的最小单位。chunk的大小可以配置，Google选择了64M。chunk设置相对大一些，对于大文件的访问有好处的：a.减少与master的通信，b.减少过长的与chunkserver保持连接，c.减少元数据对内存的占用；而对于小文件会带来集中访问的热点问题(hot spots)。

chunk副本有主角色之分，从master获得租期(lease)的副本作为主副本(primary)，租期默认的超时时间是60s，如果该chunk一直被更新，租期会一直顺延。

chunkservers也存储着相关元数据信息，主要包括两部分：chunks的checksum（占了绝大部分存储空间）、chunks的版本号。

### Client

为了减少GFS客户端与master间的频繁访问，客户端会缓存曾经访问过文件的相关信息：(filename,chunkindex)->(chunkhandle,chunklocation)，这个缓存信息在过期前或者使用的文件被重新打开前会一直有效。 

### 各组件间交互

master是路由信息的来源，client需要与master进行适当的通信获得file与chunck映射关系，这个关系会被client端缓存起来，之后会因为主副本异常或者租期被收回而失效。client端会将写入请求解耦为控制流和数据流，会先将数据通过管道方式依次拷贝到所有副本(不考虑副本角色)，然后在发送写入控制操作完成数据修改。数据流解耦的好处是可以根据网络拓扑选择最优的传输路径，最大化利用带宽，减少跨网络延时。

NOTE: GFS虽然是一个集中式架构，但是master并未成为整个系统中的瓶颈，通过将控制流与数据流分离以及通过授权租约的方式，减少客户端与master的交互请求；另外，master端存储的元数据信息也是很小的，对于内存需求也是可控的。

## 备份

snopshot

## 副本放置策略

1. 均衡chunkserver磁盘空间使用情况；
2. 控制chunkserver上同时进行拷贝副本的数量；
3. 跨多个机架来扩展副本（跨机器的优势可以最大化使用带宽、跨机架的优势在于能容忍更大的故障以及更好的扩展性）。

## 垃圾回收

被删除的文件不会被立即清理释放存储空间，而是通过重名的方式将其隐藏，通过设置保留时间来实现延迟删除。

对于孤立chunk的清理也采用延迟方式，定期通过比较master原数据信息与chunkserver上报的chunk信息，chunkserver会在空闲时将master中不存在的chunk进行清理。

延迟删除的好处如下：

1. 避免误删造成数据不可恢复性丢失；
2. 可以批量清理，提高效率减少资源消耗；
3. 在一个大的分布式系统中，信息的传输难免因各种问题(硬件问题、网络问题)，不能及时同步到所有节点，延迟可以尽可能保证信息的同步，从而保证系统的有效可靠。

## 副本过期检测

每个chunk副本都带有一个版本号，master通过维护这个版本号来比较chunk副本的新旧。收到master分配租约的chunk会将自身副本的版本号增大。

## 高可用

GFS通过快速恢复与复制机制来保证系统的高可用。

### 快速恢复

无论是master还是chunkserver，不会关心进程是正常关闭还是异常关闭，直接再重新启动即可。

master恢复时间主要是两部分，先加载元数据，然后与所有chunkservers通信获得chunk位置信息；元数通常很小加载很快，时间主要消耗在收集chunk位置。

chunkserver宕机不会对请求有任何影响，由于多副本可以保证可用，为了继续满足副本数要求，需要重建缺失的副本，通过控制复制带宽不至于影响正常应用请求，当出现副本数只剩下一个的chunk，会优先重建这些chunk。

### chunk复制

chunk通过多副本来保证可靠性，当检测到chunkserver异常，会自动进行相关chunk副本的补齐操作。

### master复制

master通过将操作日志和checkpoints复制到影子master(shadow masters)上来保证可靠性。影子master只能提供只读服务，即使当master不可用时，影子master也不允许被更新。

master是一个单一进程实现的程序，当故障后重新再起一个新的服务即可，同时将之前保存的日志复制到本地，便可以重新对外提供服务。master故障期间只是元数据处于过期的状态，但并不影响用户查询数据的实效性，用户是从chunkservers上读取数据，数据始终是最新的。

为什么影子master也需要与chunkservers进行通信，影子master的日志来源是master，自身不能跟新。

## 数据完整性

GFS不能保证每个chunk副本是完全相同的，也就无法进行副本间的互相校验，只能各自确保数据的完整性。

chunk由64KB大小的block组成，每个block都对应一个32位checksum，该checksum不和数据存储在一起，被缓存到内存中持久化到日志里面，作为chunksevers的元数据信息单独存储。查找和比较checksum时是不消耗I/O的，只有计算checksum时需要消耗I/O。

NOTE: chunk的多副本是用来解决因为chunkservers意外带来的数据不可用，chunk的checksum是用来解决由于磁盘硬件/软件驱动层问题导致数据损坏。