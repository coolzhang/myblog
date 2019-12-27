## Swap Area

**`Swap`**是对实际物理内存的补充，它扩展了进程可以使用的内存空间，同时可以将开始需要但之后不会被经常访问到的页面交换到此空间以减少了内存浪费。Swap分为两类：`Swap Partition`和`Swap File`。虚拟内存(vmsize)由RAM和Swap构成。   

### Swap操作命令  

1.创建swap  

	shell> mkswap /dev/sdc1              # 创建swap分区  
	shell> dd if=/dev/zero of=/data/swapfile bs=1M cout=1024 && mkswap /data/swapfile    # 创建swap文件  
	
2.查看swap  

	shell> swap -s
	
3.开关swap  

	shell> swapon /data/swapfile         # 开启swap  
	shell> swapoff /data/swapfile        # 关闭swap 

### 内存及Swap使用情况查看  
  
1.查看当前swap使用情况    
	
	shell> free -m

2.查看当前swap活动情况（si，so）  

	shell> vmstat 2

3.查看每个进程swap使用情况  

	shell> top (shift+f -> p)             # 按swap大小排序   
	shell> smem -u                        # 按用户统计，可以通过yum安装

4.查看指定进程swap使用情况  

	shell> grep --color VmSwap /proc/$(pidof mysqld)/status

5.控制swap使用趋势  

	shell> sysctl -w vm.swappiness=0      # 尽可能少的使用swap

6.加速回收缓存

	shell> sysctl -w vm.vfs_cache_pressure=10000

## 与MySQL相关的Swap问题  

系统发生swap会影响其上应用的使用性能，频繁的swap活动对于性能的影响是比较明显的。当数据库服务器发生swap时，需要确认其进程是否使用了swap，同时还要观察swap的活动情况。Swap的大幅波动也会引起磁IO的变化。以下情况多为引起swap的常见原因：

1.NUMA架构引起系统发生Swap；

```
shell> numactl --interleave all mysqld                  # 关闭numa   
shell> less /proc/$(pidof mysqld)/numa_maps             # 第二列为：interleave:0-1，说明numa已关闭
```
2.大量连接导致内存消耗过多触发Swap交换；   
3.批量更新、大数量查询也容易导致Swap发生。  

## 相关文章  

* [Jeremy Cole的Swap问题分析](https://blog.jcole.us/2010/09/28/mysql-swap-insanity-and-the-numa-architecture/)  
* [Improved NUMA support](https://www.percona.com/doc/percona-server/5.6/performance/innodb_numa_support.html)
* [lulu的numa解读](http://cenalulu.github.io/linux/numa/)
* [/proc/sys/vm](https://www.kernel.org/doc/Documentation/sysctl/vm.txt)
* [Swap Management](https://www.kernel.org/doc/gorman/html/understand/understand014.html)
* [Linux kernel tuning for GlusterFS](https://docs.gluster.org/en/latest/Administrator%20Guide/Linux%20Kernel%20Tuning/)