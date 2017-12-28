## ProxySQL测试笔记  

### （一）读写分离功能测试

* ProxySQL相关参数绝大多数支持在线修改，立即生效需要执行命令：`LOAD MYSQL VARIABLES TO RUNTIME`；类似还有当修改了main库下的表后，也需要执行命令：`LOAD TALBE_NAME TO RUNTIME`，例如：`LOAD MYSQL USERS TO RUNTIME`，如需持久化配置变更还需执行命令：`SAVE XXX TO DISK`。  

* 读写分离的实现需要配置路由策略，修改`mysql_query_rules`表，默认的第2条策略是需要匹配`SELECT`关键字大小写的，新添加一条规则`^select`，区别如下： 

```   
mysql> select match_pattern,re_modifiers,apply from mysql_query_rules;  
  
 match_pattern          | re_modifiers | apply   
------------------------|--------------|-------  
 ^SELECT .* FOR UPDATE$ | NULL         | 1     
 ^SELECT                | NULL         | 1       
 ^select                | CASELESS     | 1       
```   

* 1主多从的配置中，建议将主库也配置到从库分组中，当从库故障或者当主从中断发生时，主库依然可以提供查询服务。ProxySQL支持权重轮询的策略，正常架构下需要将主从的权重(weight)尽量相差大些，让从库负担更多的查询；另外，为了可以感知主从延时是否发生，需要调整延时字段(max_replication_lag)，如果延时大于该字段，实例状态会发生改变。需要注意的是，由于默认参数`mysql-monitor_slave_lag_when_null`为60，所以字段`max_replication_lag`需要小于此参数，否则遇到复制中断的情况实例状态是检测不出异常的，但并不影响正常访问。  

```
mysql> select hostgroup_id,hostname,port,status,weight,max_replication_lag from mysql_servers;

 hostgroup_id | hostname  | port | status | weight | max_replication_lag 
--------------|-----------|------|--------|--------|----------------------
 0            | 10.10.8.7 | 3306 | ONLINE | 1      | 0                   
 1            | 10.10.8.7 | 3307 | ONLINE | 9999   | 30                  
 1            | 10.10.8.7 | 3306 | ONLINE | 1      | 0                   
```  

* `mysql_users`表中`transaction_persistent `字段保证事务中的select语句始终查询主库。  

```
mysql> select username,password,active,transaction_persistent from mysql_users;

 username | password | active | transaction_persistent   
----------|----------|--------|------------------------  
 dbproxy  | dbproxy  | 1      | 1                        

```  

* 实例`SHUNNED`状态判断相关参数，如下：  

```
mysql> show variables like '%shu%';

| Variable_name                | Value |  
|------------------------------|--------  
| mysql-shun_on_failures       | 5     |  
| mysql-shun_recovery_time_sec | 10    |   
```  
See more: <https://github.com/sysown/proxysql/issues/673>  

### （二）其他功能测试

（未完待续）

