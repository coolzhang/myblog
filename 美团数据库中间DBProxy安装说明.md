## DBProxy安装

### 安装config/make时所需相关软件包

> shell> yum install -y jemalloc jemalloc-devel libevent libevent-devel openssl openssl-devel lua lua-devel bison flex libtool.x86_64 libffi libffi-devel pkg-config gettext 

### 安装MySQL相关库

> shell> yum install -y http://www.percona.com/downloads/percona-release/redhat/0.1-4/percona-release-0.1-4.noarch.rpm  
 
> shell> yum install -y Percona-Server-client-55 Percona-Server-devel-55 Percona-Server-shared-55  


### 安装glib-2.0(>=glib-2.35)

> shell> rpm -ivh **--prefix=/usr/local/glib2** glib2-2.42.0-1.el6.x86\_64.rpm glib2-devel-2.42.0-1.el6.x86\_64.rpm   

**注：**由于系统下已经安装glib(版本比较低)，glib属于基础库，被很多软件所依赖，无法升级或者重新卸载安装，故指定新目录安装满足需求的版本。


> shell> sed -i 's/\/usr\/lib64/\/usr\/local\/glib2\/lib64/g' /usr/local/glib2/lib64/*.la  

**注：**解决libtool出现大量moved提示问题    

```
libtool: link: warning: library \`/usr/local/glib2/lib64/libgthread-2.0.la' was moved.   
libtool: link: warning: library \`/usr/local/glib2/lib64/libgmodule-2.0.la' was moved.    
libtool: link: warning: library \`/usr/local/glib2/lib64/libglib\-2.0.la' was moved.   
libtool: link: warning: library \`/usr/local/glib2/lib64/libgthread-2.0.la' was moved.  
```

### 源码安装DBProxy  

> shell> git clone https://github.com/Meituan-Dianping/DBProxy.git  
> shell> cd DBProxy  
> shell> sh autogen.sh; sh bootstrap.sh  
> shell> make && make install

**注：**需要修改bootstrap.sh  

```
#!/bin/sh
base=$(cd "$(dirname "$0")"; pwd)
cd $base
./configure --prefix=/usr/local/mysql-proxy CFLAGS="-s -O0" \
GLIB_CFLAGS="-I/usr/local/glib2/include/glib-2.0 -I/usr/local/glib2/lib64/glib-2.0/include" \
GLIB_LIBS="-L/usr/local/glib2/lib64 -lglib-2.0" \
GMODULE_CFLAGS="-pthread -I/usr/local/glib2/include/glib-2.0 -I/usr/local/glib2/lib64/glib-2.0/include" \
GMODULE_LIBS="-L/usr/local/glib2/lib64 -Wl,--export-dynamic -lgmodule-2.0 -pthread -lrt" \
GTHREAD_CFLAGS="-pthread -I/usr/local/glib2/include/glib-2.0 -I/usr/local/glib2/lib64/glib-2.0/include" \
GTHREAD_LIBS="-L/usr/local/glib2/lib64 -lgthread-2.0 -pthread -lrt"
  
```

### 配置启动服务  

> shell> cd /usr/local/mysql-proxy    
> shell> mkdir conf  
> shell> cp DBProxy/script/source.cnf.samples /usr/local/mysql-proxy/conf/source.cnf  
> shell> /usr/local/mysql-proxy --defaults-file=/usr/local/mysql-proxy/conf/source.cnf  

注：配置文件中选项`pwds`是proxy连接后端数据库的用户名和使用自带encrypt工具加密后的密码。  

## 编译安装相关知识  

* ldd - 是一个shell脚本，用来查看程序动态链接的外部共享库。 

```
shell> lld mysql-proxy
	linux-vdso.so.1 =>  (0x00007fff4e9c3000)
	libgthread-2.0.so.0 => /usr/local/glib2/lib64/libgthread-2.0.so.0 (0x00007fd8e62ff000)
	libevent-1.4.so.2 => /usr/lib64/libevent-1.4.so.2 (0x00007fd8e60dd000)
	libgmodule-2.0.so.0 => /usr/local/glib2/lib64/libgmodule-2.0.so.0 (0x00007fd8e5eda000)
	libglib-2.0.so.0 => /usr/local/glib2/lib64/libglib-2.0.so.0 (0x00007fd8e5ba5000)
	libpthread.so.0 => /lib64/libpthread.so.0 (0x00007fd8e5987000)
	librt.so.1 => /lib64/librt.so.1 (0x00007fd8e577f000)
	liblua-5.1.so => /usr/lib64/liblua-5.1.so (0x00007fd8e5552000)
	libdl.so.2 => /lib64/libdl.so.2 (0x00007fd8e534d000)
	libm.so.6 => /lib64/libm.so.6 (0x00007fd8e50c9000)
	libcrypto.so.10 => /usr/lib64/libcrypto.so.10 (0x00007fd8e4ce4000)
	libjemalloc.so.1 => /usr/lib64/libjemalloc.so.1 (0x00007fd8e4aaf000)
	libc.so.6 => /lib64/libc.so.6 (0x00007fd8e471b000)
	/lib64/ld-linux-x86-64.so.2 (0x00007fd8e6501000)
	libnsl.so.1 => /lib64/libnsl.so.1 (0x00007fd8e4502000)
	libresolv.so.2 => /lib64/libresolv.so.2 (0x00007fd8e42e7000)
	libz.so.1 => /lib64/libz.so.1 (0x00007fd8e40d1000)
```  

* ldconfig - 是一个配置共享库动态链接绑定的管理命令，为了使共享接库可以被系统中需要的程序所调用，当安装共享库文件之后，需要执行ldconfig来更新`/etc/ld.so.cache`缓存文件，非默认路径需要添加到`/etc/ld.so.conf`文件中，并执行该命令。相关环境变量：`LD_LIBRARY_PATH`

```
shell> ldconfig -v    # 输出做了动态链接绑定的共享库信息

``` 

* pkg-config - 该命令通过库提供的一个`.pc`文件获得库的各种必要信息的，包括版本信息、编译和连接需要的参数等。这些信息可以通过pkg-config提供的参数单独提取出来直接供编译器和连接器使用。相关环境变量：`PKG_CONFIG_PATH`  

```
shell> pkg-config --cflags --libs glib-2.0   # cflags参数可以给出在编译时所需要的选项，libs参数可以给出连接时的选项  

```


#### 相关文章  

* [Linux的ldconfig和ldd用法](https://www.cnblogs.com/kex1n/p/5993439.html)   
* [详解pkg-config --cflags --libs glib-2.0的作用](http://blog.163.com/hu_cuit/blog/static/12284914320117140617546/)  
* [pkg-config --cflags --libs gtk+-2.0的意思、作用](http://blog.csdn.net/wangyawen0305/article/details/7936483)  
* [Shared Library Search Paths](https://www.eyrie.org/~eagle/notes/rpath.html)  
* [Shared Libraries](http://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html)  

