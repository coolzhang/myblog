## oracle日常操作命令  

### 1. 设置sqlplus默认配置  
```
SHELL> vim $ORACLE_HOME/sqlplus/admin/glogin.sql  
set linesize 200
set pagesize 60
col value for a80
col name for a60

# 备注：如果字段值以###或者科学计数展示，可以通过配置col col_name for【指定一个足够大的数字】，即可显示真实的值。
```

### 2. 日志文件的位置  
```
方法一：  
SQL> show parameter dump  

方法二：
SQL> select * from v$diag_info;   

方法三：
SHELL> adrci
adrci> show home
adrci> set homepath xxx

adrci> show alert -tail 20
adrci> show problem
adrci> show incident

```

### 3. 查看数据库相关信息  
```
-- 各种名字
SQL> show parameter name
-- 数据库版本
SQL> select * from v$version;
-- 数据库状态
SQL> select database_role,open_mode from v$database;
-- 内存使用情况
SQL> select * from v$sgainfo;
-- 查看ORL和SRL日志信息
SQL> select group#,type,member from v$logfile;

```

### 4. 查看归档信息  
```
-- 开启归档
SQL> shutdown immediate
SQL> startup mount
SQL> alter database archivelog;
SQL> alter database open;

-- 查看归档信息
SQL> archive log list

-- 归档目录
SQL> show parameter log_archive_dest

-- 查看归档文件
SQL> select name from v$archived_log;

-- 统计每天日志量
SQL> select trunc(first_time) "Date", trunc(sum(blocks*block_size)/1024/1024/1024,2) "Size(gb/day)" from v$archived_log  group by trunc(first_time)  order by 1 desc;

-- 切换日志
SQL> alter system switch logfile;

-- 使用RMAN清理归档日志
RMAN> list archivelog all;
RMAN> crosscheck arvhivelog all;
RMAN> list expired archivelog all;
RMAN> delete archivelog all completed before 'sysdate-1';

RMAN> catalog start with '/opt/oracle/arch/MYHCMPRD02/';  -- 如果rman之前不识别归档文件，可以将其注册到rman中
```

### 5. 查看DG复制状态  
```
-- 查看备库信息
-- https://docs.oracle.com/cd/E18283_01/server.112/e17110/dynviews_1096.htm
SQL> select db_unique_name,database_role,open_mode,protection_mode,force_logging,switchover_status from v$database;

-- 查看复制进程状态
SQL> select process,status,sequence# from v$managed_standby;

-- 查看日志应用情况
SQL> select name,SEQUENCE#,archived,applied from v$archived_log order by sequence#;

-- 查看是否断档
SQL> select * from v$archive_gap;

-- 查看DG状态信息
SQL> select to_char(timestamp,'DD-mm-YYYY HH24:MI:SS') time,message from v$dataguard_status;

-- 查看主库最大sequence#
SQL> select max(sequence#) from v$log;

-- 查看主库归档位置是否有错误
SQL> select error from v$archive_dest where target='STANDBY';

-- 查看主备上DG日志
SQL> select * from v$dataguard_status;


```

### 6a. DG物理备库进行switchover  
```
-- 11g切换步骤
Primary-SQL> ALTER DATABASE COMMIT TO SWITCHOVER TO PHYSICAL STANDBY WITH SESSION SHUTDOWN;  # 12c开始可以不加WITH SESSION SHUTDOWN
Standby-SQL> ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY WITH SESSION SHUTDOWN;
# 参考：https://docs.oracle.com/database/121/SBYDB/pre12_role_trans.htm#SBYDB5173

-- 12c切换步骤
Primary-SQL> ALTER DATABASE SWITCHOVER TO EBSPRD_P VERIFY;  # 检查是否可以切换
Primary-SQL> ALTER DATABASE SWITCHOVER TO EBSPRD_P;
# 参考：https://docs.oracle.com/en/database/oracle/oracle-database/19/spmss/switchover-to-a-physical-db.html#GUID-AAD70601-D248-4309-B8DD-F461EE31A5FF
```

### 6b. DG物理备库进行failover  
```
-- 11g切换步骤
Standby-SQL> ALTER DATABASE RECOVERY MANAGED STANDBY DATABASE FINISH;
Standby-SQL> ALTER DATABASE ACTIVATE STANDBY DATABASE;
# 参考：http://oracle-help.com/dataguard/manual-failover-in-data-guard/
```

### 7. 配置多个物理备库  
```
Primary-SQL> alter system set LOG_ARCHIVE_CONFIG='DG_CONFIG=(MYHCMDV0,MYHCMDV1,MYHCMDV2)';  
Primary-SQL> alter system set log_archive_dest_3='SERVICE=MYHCMPRD_S2 LGWR ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=MYHCMDV2';
Primary-SQL> alter system set log_archive_dest_state_3='enable';  # 默认enable  
```

### 8. 查看用户信息  
```
-- 查看用户密码
SQL> select username,password_verison from dba_users where username='sys';
SQL> select name,password,SPARE4 from user$ where name='sys';

-- 查看当前用户有哪些角色
SQL> select * from user_role_privs;

-- 查看指定角色包含哪些权限
SQL> select * from role_sys_privs where role in ('CONNECT', 'RESOURCE') order by role;

-- 查看当前用户有哪些权限
SQL> select * from session_privs;
```

### 9. UNDO表空间  
```
-- 查看UNDO信息
SQL> show parameter undo
SQL> select tablespace_name,retention from dba_tablespaces where contents='UNDO';
SQL> select file_name,tablespace_name,bytes from dba_data_files where tablespace_name='APPS_UNDOTS1';
-- 新建UNDO表空间
SQL> create undo tablespace NEW_UNDOTS01 add datafile '/u01/oradata/undtbs01.dbf' size 10M autoextend on;
-- 切换UNDO表空间
SQL> alter system set undo_tablespace='NEW_UNDOTS01';
-- 修改retention
SQL> alter tablespace NEW_UNDOTS01 retention [no]guarantee;
-- 查看UNDO使用情况
SQL> select status,sum(bytes)/1024/1024 from dba_undo_extents group by status;

```

### 10. 临时表空间
```
-- 查询用户对应的表空间信息
SQL> select USERNAME,DEFAULT_TABLESPACE,TEMPORARY_TABLESPACE from dba_users where USERNAME='CUX_FUND';
-- 查询临时表空间数据文件大小
SQL>  select FILE_NAME,TABLESPACE_NAME,BYTES/1024/1024/1024 as size_GB from dba_temp_files where TABLESPACE_NAME='CUX_TEMP';
-- 查询临时表空间使用率
SQL> 
SELECT D.tablespace_name,
       SPACE "SUM_SPACE(M)",
       blocks "SUM_BLOCKS",
       used_space "USED_SPACE(M)",
       Round(Nvl(used_space, 0) / SPACE * 100, 2) "USED_RATE(%)",
       SPACE - used_space "FREE_SPACE(M)"
 FROM (SELECT tablespace_name,
               Round(SUM(bytes) / (1024 * 1024), 2) SPACE,
               SUM(blocks) BLOCKS
          FROM dba_temp_files
         GROUP BY tablespace_name) D,
       (SELECT tablespace,
               Round(SUM(blocks * 8192) / (1024 * 1024), 2) USED_SPACE
          FROM v$sort_usage
         GROUP BY tablespace) F
 WHERE D.tablespace_name = F.tablespace(+) 
 AND D.tablespace_name in ('TEMP', 'CUX_TEMP');
-- 新增临时表空间文件
SQL> alter tablespace CUX_TEMP add tempfile '/opt/oracle/oradata/ERPFIN/cux_temp02.dbf' size 30G;
SQL> alter database tempfile '/opt/oracle/oradata/ERPFIN/cux_temp02.dbf' autoextend on next 100m maxsize unlimited;

```


### 11. RMAN相关   
```
-- 查看备份进度
SQL> select case when opname like  '%aggregate%'  then 'total' else opname  end opname,trunc(sofar*100/totalwork,2)||'%' progress, units from  v$session_longops where opname like 'RMAN%' and totalwork>sofar;  

```

### 12. AWR相关  
```
-- 查看AWR配置
SQL> select dbid,retention from dba_hist_wr_control;  

-- 查看snapshot信息  
SQL> select min(snap_id), max(snap_id) from dba_hist_snapshot;

```

### 13. 查询当前进程  
```
-- 查看当前正在执行的SQL
SQL> SELECT b.sid,b.serial#,b.username,sql_text,b.machine FROM v$process a, v$session b,v$sqlarea c WHERE a.addr = b.paddr and b.sql_hash_value = c.hash_value and b.username='CUX_FUND';

-- 查  
SQL> ;

```


### 常见错误
```
-- 查看错误原因及解决办法
SQL> !oerr ora N
```
1. 问题：**ORA-21561: OID generation failed**   
解决：检查/etc/hosts配置是否正确。
2. 问题：**备库没有RFS进程，MRP进程总是WAIT_FOR_LOG状态，SEQUECE#不主动变化**   
解决：检查主库log\_archive\_dest\_2配置中service选项是否正确；密码文件是否一致（另外，排除下是否开启了防火墙）。
3. 问题：启动时提示**Oracle system identifier already exists specify another SID**  
解决：删除/etc/oratab中对应SID记录，或者$ORACLE_BASE/oraInventory/ContentsXML/inventory.xml中存在相同SID记录。
4. 问题：主备切换后，打开新主库时提示**RA-30012: undo tablespace 'UNDOTBS1' does not exist or of wrong type**  
解决：检查参数undo\_tablespace=‘APPS_UNDOTS1’是否与原主库配置一致。
5. 问题：**ORA-16484: compatibility setting is too low**（执行select ERROR from v$archive_dest提示） 导致复制异常  
解决：备库配置参数compatible='12.1.0.2' 不能低于主库 
6. 问题：通过rman duplicate搭建好物理机备库，如果直接alter database open，会报如下错误： 
**ORA-10458: standby database requires recovery  
ORA-01152: file 1 was not restored from a sufficiently old backup  
ORA-01110: data file 1: '/opt/oracle/oradata/MYHCMDV0/system01.dbf'**  
解决：首先需要保证主备复制是通的，rman duplicate结束后需要等待将备份期间产生的归档日志文档全部同步到备库后(不能有GAP，否则不能执行redo apply)，再执行`ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT FROM SESSION;` ，然后执行`ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;`，最后再打开数据库。
7. 问题：**ORA-16047: DGID mismatch between destination setting and target database**  
解决： 首先确认主备`log_archive_config`以及备库`db_unique_name`配置是否正确，然后在主库重新对参数`log_archive_dest_state_2`先设置为defer，再设置为enable，备库复制进程恢复正常。  
8. 问题： 业务一条统计SQL，通过客户端执行正常返回，Java程序执行时报错，**ORA-01652: unable to extend temp segment by 128 in tablespace CUX_TEMP**  
解决：重启oracle（根据报错，尝试增加临时表空间大小，问题依然存在，从30G扩容至60G，该临时表空间使用率81%）			