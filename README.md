# deploy_proxysql-MGR57
参考文档：
https://github.com/sysown/proxysql/wiki#getting-started

https://github.com/sysown/proxysql
## 零、MySQL准备
### 0.1搭建MGR
主节点运行如下sql，sys库下创建视图，用于proxysql监控MGR：
```sql
USE sys;

DELIMITER $$

CREATE FUNCTION IFZERO(a INT, b INT)
RETURNS INT
DETERMINISTIC
RETURN IF(a = 0, b, a)$$

CREATE FUNCTION LOCATE2(needle TEXT(10000), haystack TEXT(10000), offset INT)
RETURNS INT
DETERMINISTIC
RETURN IFZERO(LOCATE(needle, haystack, offset), LENGTH(haystack) + 1)$$

CREATE FUNCTION GTID_NORMALIZE(g TEXT(10000))
RETURNS TEXT(10000)
DETERMINISTIC
RETURN GTID_SUBTRACT(g, '')$$

CREATE FUNCTION GTID_COUNT(gtid_set TEXT(10000))
RETURNS INT
DETERMINISTIC
BEGIN
  DECLARE result BIGINT DEFAULT 0;
  DECLARE colon_pos INT;
  DECLARE next_dash_pos INT;
  DECLARE next_colon_pos INT;
  DECLARE next_comma_pos INT;
  SET gtid_set = GTID_NORMALIZE(gtid_set);
  SET colon_pos = LOCATE2(':', gtid_set, 1);
  WHILE colon_pos != LENGTH(gtid_set) + 1 DO
     SET next_dash_pos = LOCATE2('-', gtid_set, colon_pos + 1);
     SET next_colon_pos = LOCATE2(':', gtid_set, colon_pos + 1);
     SET next_comma_pos = LOCATE2(',', gtid_set, colon_pos + 1);
     IF next_dash_pos < next_colon_pos AND next_dash_pos < next_comma_pos THEN
       SET result = result +
         SUBSTR(gtid_set, next_dash_pos + 1,
                LEAST(next_colon_pos, next_comma_pos) - (next_dash_pos + 1)) -
         SUBSTR(gtid_set, colon_pos + 1, next_dash_pos - (colon_pos + 1)) + 1;
     ELSE
       SET result = result + 1;
     END IF;
     SET colon_pos = next_colon_pos;
  END WHILE;
  RETURN result;
END$$

CREATE FUNCTION gr_applier_queue_length()
RETURNS INT
DETERMINISTIC
BEGIN
  RETURN (SELECT sys.gtid_count( GTID_SUBTRACT( (SELECT
Received_transaction_set FROM performance_schema.replication_connection_status
WHERE Channel_name = 'group_replication_applier' ), (SELECT
@@global.GTID_EXECUTED) )));
END$$

CREATE FUNCTION gr_member_in_primary_partition()
RETURNS VARCHAR(3)
DETERMINISTIC
BEGIN
  RETURN (SELECT IF( MEMBER_STATE='ONLINE' AND ((SELECT COUNT(*) FROM
performance_schema.replication_group_members WHERE MEMBER_STATE != 'ONLINE') >=
((SELECT COUNT(*) FROM performance_schema.replication_group_members)/2) = 0),
'YES', 'NO' ) FROM performance_schema.replication_group_members JOIN
performance_schema.replication_group_member_stats USING(member_id));
END$$

CREATE VIEW gr_member_routing_candidate_status AS SELECT
sys.gr_member_in_primary_partition() as viable_candidate,
IF( (SELECT (SELECT GROUP_CONCAT(variable_value) FROM
performance_schema.global_variables WHERE variable_name IN ('read_only',
'super_read_only')) != 'OFF,OFF'), 'YES', 'NO') as read_only,
sys.gr_applier_queue_length() as transactions_behind, Count_Transactions_in_queue as 'transactions_to_cert' from performance_schema.replication_group_member_stats;$$

DELIMITER ;
```
上述视图可以用来监控从库延时情况，如下为压测过程中从库事务延时情况
```sql
mysql> select * from sys.gr_member_routing_candidate_status;
+------------------+-----------+---------------------+----------------------+
| viable_candidate | read_only | transactions_behind | transactions_to_cert |
+------------------+-----------+---------------------+----------------------+
| YES              | YES       |                   3 |                    2 |
+------------------+-----------+---------------------+----------------------+
1 row in set (0.00 sec)
```
primary节点如下
```sql
mysql> select * from sys.gr_member_routing_candidate_status;
+------------------+-----------+---------------------+----------------------+
| viable_candidate | read_only | transactions_behind | transactions_to_cert |
+------------------+-----------+---------------------+----------------------+
| YES              | NO        |                   0 |                    0 |
+------------------+-----------+---------------------+----------------------+
1 row in set (0.00 sec)
```

## 一、安装proxysql，操作系统centos7

### 1.1下载安装包
```sh
wget https://github.com/sysown/proxysql/releases/download/v2.0.12/proxysql-2.0.12-1-centos7.x86_64.rpm
```
proxysql-2.0.12-1-centos7.x86_64.rpm
### 1.2 安装依赖，安装proxysql
```sh
sudo yum install gnutls
sudo yum install https://repo.percona.com/yum/percona-release-latest.noarch.rpm
yum install percona-xtrabackup-24

rpm -ivh proxysql-2.0.12-1-centos7.x86_64.rpm
```
## 二、启动proxysql
```sh
systemctl start proxysql 
```
查看状态
```sh
systemctl status proxysql
[root@dzst140 ~]# systemctl status proxysql
● proxysql.service - High Performance Advanced Proxy for MySQL
   Loaded: loaded (/etc/systemd/system/proxysql.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2020-05-27 19:14:59 CST; 18h ago
  Process: 47492 ExecStart=/usr/bin/proxysql --idle-threads -c /etc/proxysql.cnf $PROXYSQL_OPTS (code=exited, status=0/SUCCESS)
 Main PID: 47494 (proxysql)
   CGroup: /docker/ee78902be09220304571c4c16bed1f2db0d2353b78f61e1747ec83531a420f59/system.slice/proxysql.service
           ├─47494 /usr/bin/proxysql --idle-threads -c /etc/proxysql.cnf
           └─47495 /usr/bin/proxysql --idle-threads -c /etc/proxysql.cnf
           ‣ 47494 /usr/bin/proxysql --idle-threads -c /etc/proxysql.cnf

May 27 19:14:59 dzst140 systemd[1]: Starting High Performance Advanced Proxy for MySQL...
May 27 19:14:59 dzst140 proxysql[47492]: 2020-05-27 19:14:59 [INFO] Using config file /etc/proxysql.cnf
May 27 19:14:59 dzst140 proxysql[47492]: 2020-05-27 19:14:59 [INFO] Using OpenSSL version: OpenSSL 1.1.1d  10 Sep 2019
May 27 19:14:59 dzst140 proxysql[47492]: 2020-05-27 19:14:59 [INFO] No SSL keys/certificates found in datadir (/var/lib/proxysql). Generating new keys/certificates.
May 27 19:14:59 dzst140 systemd[1]: Started High Performance Advanced Proxy for MySQL.
```
proxysql默认状态开启两个端口

6032管理端口

6033服务端口(应用连接该端口，可也进入管理界面更改该配置为指定端口，但是需要重启proxysql服务)

## 三、进入管理配置界面
proxysql配置参数分为如下三个层面：DISK MEMORY RUNTIME
```sql
+-------------------------+
|         RUNTIME         |
+-------------------------+
       /|\          |
        |           |
    [1] |       [2] |
        |          \|/
+-------------------------+
|         MEMORY          |
+-------------------------+ _
       /|\          |      |\
        |           |        \
    [3] |       [4] |         \ [5]
        |          \|/         \
+-------------------------+  +-------------------------+
|          DISK           |  |       CONFIG FILE       |
+-------------------------+  +-------------------------+
```
执行如下命令进入管理界面，proxysql的管理类似于MySQL，可以通过更改变量，更改表中数据对proxysql进行配置
```sh
mysql -u admin  -padmin  -h127.0.0.1 -P6032 --prompt='Admin>'

mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 6
Server version: 5.7.30 (ProxySQL Admin Module)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement
```
```sql
Admin>show databases;
+-----+---------------+-------------------------------------+
| seq | name          | file                                |
+-----+---------------+-------------------------------------+
| 0   | main          |                                     |
| 2   | disk          | /var/lib/proxysql/proxysql.db       |
| 3   | stats         |                                     |
| 4   | monitor       |                                     |
| 5   | stats_history | /var/lib/proxysql/proxysql_stats.db |
+-----+---------------+-------------------------------------+
5 rows in set (0.00 sec)
```
使用show tables;命令可以查看核心的配置表

runtime_开头的表为当前生效(RUNTIME)的配置表

其他mysql_开头表为MEMORY状态的表(非生效参数)
```sql
Admin> show tables;
+----------------------------------------------------+
| tables                                             |
+----------------------------------------------------+
| global_variables                                   |
| mysql_aws_aurora_hostgroups                        |
| mysql_collations                                   |
| mysql_firewall_whitelist_rules                     |
| mysql_firewall_whitelist_sqli_fingerprints         |
| mysql_firewall_whitelist_users                     |
| mysql_galera_hostgroups                            |
| mysql_group_replication_hostgroups                 |
| mysql_query_rules                                  |
| mysql_query_rules_fast_routing                     |
| mysql_replication_hostgroups                       |
| mysql_servers                                      |
| mysql_users                                        |
| proxysql_servers                                   |
| restapi_routes                                     |
| runtime_checksums_values                           |
| runtime_global_variables                           |
| runtime_mysql_aws_aurora_hostgroups                |
| runtime_mysql_firewall_whitelist_rules             |
| runtime_mysql_firewall_whitelist_sqli_fingerprints |
| runtime_mysql_firewall_whitelist_users             |
| runtime_mysql_galera_hostgroups                    |
| runtime_mysql_group_replication_hostgroups         |
| runtime_mysql_query_rules                          |
| runtime_mysql_query_rules_fast_routing             |
| runtime_mysql_replication_hostgroups               |
| runtime_mysql_servers                              |
| runtime_mysql_users                                |
| runtime_proxysql_servers                           |
| runtime_restapi_routes                             |
| runtime_scheduler                                  |
| scheduler                                          |
+----------------------------------------------------+
32 rows in set (0.00 sec)
```
查看核心配置表：

mysql_servers及mysql_group_replication_hostgroups(MGR需要配置该表)表结构
```sql
Admin> show create table mysql_servers\G
*************************** 1. row ***************************
       table: mysql_servers
Create Table: CREATE TABLE mysql_servers (
    hostgroup_id INT CHECK (hostgroup_id>=0) NOT NULL DEFAULT 0,
    hostname VARCHAR NOT NULL,
    port INT CHECK (port >= 0 AND port <= 65535) NOT NULL DEFAULT 3306,
    gtid_port INT CHECK ((gtid_port <> port OR gtid_port=0) AND gtid_port >= 0 AND gtid_port <= 65535) NOT NULL DEFAULT 0,
    status VARCHAR CHECK (UPPER(status) IN ('ONLINE','SHUNNED','OFFLINE_SOFT', 'OFFLINE_HARD')) NOT NULL DEFAULT 'ONLINE',
    weight INT CHECK (weight >= 0 AND weight <=10000000) NOT NULL DEFAULT 1,
    compression INT CHECK (compression IN(0,1)) NOT NULL DEFAULT 0,
    max_connections INT CHECK (max_connections >=0) NOT NULL DEFAULT 1000,
    max_replication_lag INT CHECK (max_replication_lag >= 0 AND max_replication_lag <= 126144000) NOT NULL DEFAULT 0,
    use_ssl INT CHECK (use_ssl IN(0,1)) NOT NULL DEFAULT 0,
    max_latency_ms INT UNSIGNED CHECK (max_latency_ms>=0) NOT NULL DEFAULT 0,
    comment VARCHAR NOT NULL DEFAULT '',
    PRIMARY KEY (hostgroup_id, hostname, port) )
1 row in set (0.00 sec)

Admin>show create table mysql_group_replication_hostgroups\G
*************************** 1. row ***************************
       table: mysql_group_replication_hostgroups
Create Table: CREATE TABLE mysql_group_replication_hostgroups (
    writer_hostgroup INT CHECK (writer_hostgroup>=0) NOT NULL PRIMARY KEY,
    backup_writer_hostgroup INT CHECK (backup_writer_hostgroup>=0 AND backup_writer_hostgroup<>writer_hostgroup) NOT NULL,
    reader_hostgroup INT NOT NULL CHECK (reader_hostgroup<>writer_hostgroup AND backup_writer_hostgroup<>reader_hostgroup AND reader_hostgroup>0),
    offline_hostgroup INT NOT NULL CHECK (offline_hostgroup<>writer_hostgroup AND offline_hostgroup<>reader_hostgroup AND backup_writer_hostgroup<>offline_hostgroup AND offline_hostgroup>=0),
    active INT CHECK (active IN (0,1)) NOT NULL DEFAULT 1,
    max_writers INT NOT NULL CHECK (max_writers >= 0) DEFAULT 1,
    writer_is_also_reader INT CHECK (writer_is_also_reader IN (0,1,2)) NOT NULL DEFAULT 0,
    max_transactions_behind INT CHECK (max_transactions_behind>=0) NOT NULL DEFAULT 0,
    comment VARCHAR,
    UNIQUE (reader_hostgroup),
    UNIQUE (offline_hostgroup),
    UNIQUE (backup_writer_hostgroup))
1 row in set (0.01 sec)

```
## 四、配置MySQL复制集群中的IP及端口，默认hostgroup_id都设置为1
将MGR中的节点信息写入mysql_servers表中
```sql
INSERT INTO mysql_servers(hostgroup_id, hostname, port) VALUES (1,'172.18.0.151',3317);
INSERT INTO mysql_servers(hostgroup_id, hostname, port) VALUES (1,'172.18.0.152',3317);
INSERT INTO mysql_servers(hostgroup_id, hostname, port) VALUES (1,'172.18.0.160',3317);
```
查看结果
```sql
Admin> select * from mysql_servers;
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname     | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 1            | 172.18.0.151 | 3317 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 1            | 172.18.0.152 | 3317 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 1            | 172.18.0.160 | 3317 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
```
将结果生效，并保存的磁盘
```sql 
load mysql servers to run;
save mysql servers to disk;
```

## 五、设置proxysql监控mysql的用户。

MySQL集群中创建如下用户：
```sql 
create user monitor identified by "monitor";
grant select on sys.* to monitor;
```
proxyql默认监控用户名和密码如下。
```sql 
set mysql-monitor_username='monitor';
set mysql-monitor_password='monitor';
```
将设置生效
```sql
Admin>load mysql variables to runtime;
```
将配置持久化到硬盘
```sql 
Admin>save mysql variables to disk;
```
## 六、查看监控日志，在monitor用户增加完成后，connect_error为NULL,表示配置正常。
```sql 
Admin> select * from mysql_server_connect_log;
+--------------+------+------------------+-------------------------+---------------+
| hostname     | port | time_start_us    | connect_success_time_us | connect_error |
+--------------+------+------------------+-------------------------+---------------+
| 172.18.0.152 | 3317 | 1590582101880120 | 559                     | NULL          |
| 172.18.0.151 | 3317 | 1590582160526822 | 675                     | NULL          |
| 172.18.0.160 | 3317 | 1590582161033245 | 660                     | NULL          |
| 172.18.0.152 | 3317 | 1590582161539608 | 573                     | NULL          |
| 172.18.0.151 | 3317 | 1590582220526996 | 442                     | NULL          |
| 172.18.0.152 | 3317 | 1590582221105073 | 637                     | NULL          |
| 172.18.0.160 | 3317 | 1590582221683261 | 642                     | NULL          |
| 172.18.0.152 | 3317 | 1590582280526936 | 1210                    | NULL          |
```

## 七、下表将监控MySQL实例是否alive.
```sql 
Admin> select * from mysql_server_ping_log limit 10;
+--------------+------+------------------+----------------------+------------+
| hostname     | port | time_start_us    | ping_success_time_us | ping_error |
+--------------+------+------------------+----------------------+------------+
| 172.18.0.152 | 3317 | 1590582101062211 | 133                  | NULL       |
| 172.18.0.152 | 3317 | 1590582110799191 | 211                  | NULL       |
| 172.18.0.160 | 3317 | 1590582110921015 | 121                  | NULL       |

| 172.18.0.160 | 3317 | 1590582130799374 | 193                  | NULL       |
| 172.18.0.152 | 3317 | 1590582130892324 | 159                  | NULL       |
| 172.18.0.151 | 3317 | 1590582130985287 | 197                  | NULL       |
+--------------+------+------------------+----------------------+------------+
10 rows in set (0.01 sec)
```

## 八、查看mysql_server_group_replication_log此时为空
该表中记录monitor用户查询sys.gr_member_routing_candidate_status视图的结果，

需要配置mysql_group_replication_hostgroups后，mysql_server_group_replication_log才会写入信息。

## 九、配置MGR集群分组表：mysql_group_replication_hostgroups

proxysql通过监控后台sys.gr_member_routing_candidate_status视图（必须创建）;proxysql将节点分组

配置MGR分组信息
```sql
insert into  mysql_group_replication_hostgroups(writer_hostgroup,backup_writer_hostgroup,reader_hostgroup,offline_hostgroup) values(1,2,3,4);

Admin>load mysql servers to runtime;

Admin>save mysql servers to disk;

Admin> select * from mysql_group_replication_hostgroups;
+------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
| writer_hostgroup | backup_writer_hostgroup | reader_hostgroup | offline_hostgroup | active | max_writers | writer_is_also_reader | max_transactions_behind | comment |
+------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
| 1                | 2                       | 3                | 4                 | 1      | 1           | 0                     | 0                       | NULL    |
+------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
```
注：writer_is_also_reader=1时primary节点也会有读负载。

为了保证primary节点和secondary节点读到一致的数据。

max_transactions_behind=N参数配置后，如果从节点事务延时个数超过N后（mysql_server_group_replication_log中会有记录）

secondary节点将不再有读负载，从reader_hostgroup组中移除，

直到延时事务个数降到N以下，节点重新加入reader_hostgroup组。

默认状态下

## 十、配置生效后查看状态，会对MySQLserver进行分组
可以观察到160和151被分配到hostgroup_id=3的组中，即reader_hostgroup

可以更新mysql_servers表中的weight值，对读负载量进行配置

查看当前生效状态的runtime_mysql_servers表，

此时proxysql对MGR中的不同节点进行重新分组

primary节点172.18.0.152 分配到hostgroup_id=1的组，即writer_hostgroup

secondary节点172.18.0.151/160 分配到hostgroup_id=3的组，即reader_hostgroup
```sql 
Admin> select * from runtime_mysql_servers;
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname     | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 1            | 172.18.0.152 | 3317 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 3            | 172.18.0.160 | 3317 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 3            | 172.18.0.151 | 3317 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
```

## 十一、查看mysql_server_group_replication_log监控状态：
使用如下命令可以看到MGR节点的状态，其中只有主节点read_only为NO
```sql 
Admin> select * from mysql_server_group_replication_log limit 3;
+--------------+------+------------------+-----------------+------------------+-----------+---------------------+-------+
| hostname     | port | time_start_us    | success_time_us | viable_candidate | read_only | transactions_behind | error |
+--------------+------+------------------+-----------------+------------------+-----------+---------------------+-------+
| 172.18.0.151 | 3317 | 1590582205474364 | 2999            | YES              | YES       | 0                   | NULL  |
| 172.18.0.152 | 3317 | 1590646316421808 | 2018            | YES              | NO        | 0                   | NULL  |
| 172.18.0.160 | 3317 | 1590646201420699 | 3063            | YES              | YES       | 0                   | NULL  |
+--------------+------+------------------+-----------------+------------------+-----------+---------------------+-------+
```

## 十二、创建用户，配置基于用户的读写分离
MGR中创建用户
```sql 
create user reader identified by 'passwd';
grant select on *.* to reader with grant option;

insert into mysql_users(username,password,active,default_hostgroup) values("tian","passwd",1,1) ;
insert into mysql_users(username,password,active,default_hostgroup) values("reader","passwd",1,3) ;

load mysql users to run;
save mysql users to disk;


Admin>insert into mysql_users(username,password,active,default_hostgroup) values("tian","passwd",1,1) ;
Query OK, 1 row affected (0.00 sec)

Admin>insert into mysql_users(username,password,active,default_hostgroup) values("reader","passwd",1,3) ;
Query OK, 1 row affected (0.00 sec)

Admin>load mysql users to run;
Query OK, 0 rows affected (0.00 sec)

Admin>save mysql users to disk;
Query OK, 0 rows affected (0.00 sec)

查看runtime状态下的用户及分组
Admin>select * from runtime_mysql_users;
+----------+-------------------------------------------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+---------+
| username | password                                  | active | use_ssl | default_hostgroup | default_schema | schema_locked | transaction_persistent | fast_forward | backend | frontend | max_connections | comment |
+----------+-------------------------------------------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+---------+
| tian     | *E66F04FE5F167BF5F78CB239C283CDCD51A2E33F | 1      | 0       | 1                 |                | 0             | 1                      | 0            | 0       | 1        | 10000           |         |
| reader   | *E66F04FE5F167BF5F78CB239C283CDCD51A2E33F | 1      | 0       | 3                 |                | 0             | 1                      | 0            | 0       | 1        | 10000           |         |
| tian     | *E66F04FE5F167BF5F78CB239C283CDCD51A2E33F | 1      | 0       | 1                 |                | 0             | 1                      | 0            | 1       | 0        | 10000           |         |
| reader   | *E66F04FE5F167BF5F78CB239C283CDCD51A2E33F | 1      | 0       | 3                 |                | 0             | 1                      | 0            | 1       | 0        | 10000           |         |
+----------+-------------------------------------------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+---------+
4 rows in set (0.01 sec)
```
读写分离测试

使用两个不同的用户连接proxysql可以将sql路由到不同的default_hostgroup，进而：使用写用户访问最终到primary节点，使用读用户，最终访问secondary节点。

此时主节点为152

从节点为151和160
```sh
[root@dzst140 ~]# mysql -u reader  -ppasswd  -h127.0.0.1 -P6033 -e "select @@server_id";
mysql: [Warning] Using a password on the command line interface can be insecure.
+-------------+
| @@server_id |
+-------------+
|     1513317 |
+-------------+
[root@dzst140 ~]# mysql -u reader  -ppasswd  -h127.0.0.1 -P6033 -e "select @@server_id";
mysql: [Warning] Using a password on the command line interface can be insecure.
+-------------+
| @@server_id |
+-------------+
|     1513317 |
+-------------+
[root@dzst140 ~]# mysql -u reader  -ppasswd  -h127.0.0.1 -P6033 -e "select @@server_id";
mysql: [Warning] Using a password on the command line interface can be insecure.
+-------------+
| @@server_id |
+-------------+
|     1603317 |
+-------------+
[root@dzst140 ~]# mysql -u reader  -ppasswd  -h127.0.0.1 -P6033 -e "select @@server_id";
mysql: [Warning] Using a password on the command line interface can be insecure.
+-------------+
| @@server_id |
+-------------+
|     1513317 |
+-------------+
[root@dzst140 ~]# 
```
## 十三、主节点也可读，基于sql语句的读写分离
```sql
Admin>update mysql_group_replication_hostgroups set writer_is_also_reader =1;
Query OK, 1 row affected (0.00 sec)

Admin>select * from runtime_mysql_group_replication_hostgroups;
+------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
| writer_hostgroup | backup_writer_hostgroup | reader_hostgroup | offline_hostgroup | active | max_writers | writer_is_also_reader | max_transactions_behind | comment |
+------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
| 1                | 2                       | 3                | 4                 | 1      | 1           | 0                     | 0                       | NULL    |
+------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
1 row in set (0.00 sec)

Admin>load mysql servers to run;
Query OK, 0 rows affected (0.01 sec)

Admin>select * from runtime_mysql_group_replication_hostgroups;
+------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
| writer_hostgroup | backup_writer_hostgroup | reader_hostgroup | offline_hostgroup | active | max_writers | writer_is_also_reader | max_transactions_behind | comment |
+------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
| 1                | 2                       | 3                | 4                 | 1      | 1           | 1                     | 0                       | NULL    |
+------------------+-------------------------+------------------+-------------------+--------+-------------+-----------------------+-------------------------+---------+
1 row in set (0.01 sec)
```
基于sql语句的读写分离
proxysql执行如下命令
```sql 
UPDATE mysql_users SET default_hostgroup=1; 
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK; 

INSERT INTO mysql_query_rules (rule_id,active,username,match_pattern,destination_hostgroup,apply)
VALUES(1,1,'tian','^SELECT.*FOR UPDATE$',1,1),(2,1,'tian','^SELECT',3,1);
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```
查看server 情况
```sql
Admin>select * from runtime_mysql_servers;
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname     | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 1            | 172.18.0.152 | 3317 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 3            | 172.18.0.160 | 3317 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 3            | 172.18.0.151 | 3317 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 3            | 172.18.0.152 | 3317 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
4 rows in set (0.01 sec)


update mysql_users set transaction_persistent=0; 
load mysql users to run;
save mysql users to disk;

Admin>select * from runtime_mysql_users;
+----------+-------------------------------------------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+---------+
| username | password                                  | active | use_ssl | default_hostgroup | default_schema | schema_locked | transaction_persistent | fast_forward | backend | frontend | max_connections | comment |
+----------+-------------------------------------------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+---------+
| tian     | *E66F04FE5F167BF5F78CB239C283CDCD51A2E33F | 1      | 0       | 1                 |                | 0             | 0                      | 0            | 0       | 1        | 10000           |         |
| reader   | *E66F04FE5F167BF5F78CB239C283CDCD51A2E33F | 1      | 0       | 1                 |                | 0             | 0                      | 0            | 0       | 1        | 10000           |         |
| tian     | *E66F04FE5F167BF5F78CB239C283CDCD51A2E33F | 1      | 0       | 1                 |                | 0             | 0                      | 0            | 1       | 0        | 10000           |         |
| reader   | *E66F04FE5F167BF5F78CB239C283CDCD51A2E33F | 1      | 0       | 1                 |                | 0             | 0                      | 0            | 1       | 0        | 10000           |         |
+----------+-------------------------------------------+--------+---------+-------------------+----------------+---------------+------------------------+--------------+---------+----------+-----------------+---------+
4 rows in set (0.00 sec)

Admin>select * from runtime_mysql_query_rules;
+---------+--------+----------+------------+--------+-------------+------------+------------+--------+--------------+----------------------+----------------------+--------------+---------+-----------------+-----------------------+-----------+--------------------+---------------+-----------+---------+---------+-------+-------------------+----------------+------------------+-----------+--------+-------------+-----------+---------------------+-----+-------+---------+
| rule_id | active | username | schemaname | flagIN | client_addr | proxy_addr | proxy_port | digest | match_digest | match_pattern        | negate_match_pattern | re_modifiers | flagOUT | replace_pattern | destination_hostgroup | cache_ttl | cache_empty_result | cache_timeout | reconnect | timeout | retries | delay | next_query_flagIN | mirror_flagOUT | mirror_hostgroup | error_msg | OK_msg | sticky_conn | multiplex | gtid_from_hostgroup | log | apply | comment |
+---------+--------+----------+------------+--------+-------------+------------+------------+--------+--------------+----------------------+----------------------+--------------+---------+-----------------+-----------------------+-----------+--------------------+---------------+-----------+---------+---------+-------+-------------------+----------------+------------------+-----------+--------+-------------+-----------+---------------------+-----+-------+---------+
| 1       | 1      | tian     | NULL       | 0      | NULL        | NULL       | NULL       | NULL   | NULL         | ^SELECT.*FOR UPDATE$ | 0                    | CASELESS     | NULL    | NULL            | 1                     | NULL      | NULL               | NULL          | NULL      | NULL    | NULL    | NULL  | NULL              | NULL           | NULL             | NULL      | NULL   | NULL        | NULL      | NULL                | NULL | 1     | NULL    |
| 2       | 1      | tian     | NULL       | 0      | NULL        | NULL       | NULL       | NULL   | NULL         | ^SELECT              | 0                    | CASELESS     | NULL    | NULL            | 3                     | NULL      | NULL               | NULL          | NULL      | NULL    | NULL    | NULL  | NULL              | NULL           | NULL             | NULL      | NULL   | NULL        | NULL      | NULL                | NULL | 1     | NULL    |
+---------+--------+----------+------------+--------+-------------+------------+------------+--------+--------------+----------------------+----------------------+--------------+---------+-----------------+-----------------------+-----------+--------------------+---------------+-----------+---------+---------+-------+-------------------+----------------+------------------+-----------+--------+-------------+-----------+---------------------+-----+-------+---------+
2 rows in set (0.00 sec)

Admin>select * from runtime_mysql_servers;
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| hostgroup_id | hostname     | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment |
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
| 1            | 172.18.0.152 | 3317 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 3            | 172.18.0.160 | 3317 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 3            | 172.18.0.151 | 3317 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
| 3            | 172.18.0.152 | 3317 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              |         |
+--------------+--------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------+
```

压测，查看统计
```sql 
Admin> SELECT hostgroup,digest,SUBSTR(digest_text,0,25),count_star,sum_time FROM stats_mysql_query_digest WHERE digest_text LIKE 'SELECT%' ORDER BY sum_time DESC LIMIT 100;
+-----------+--------------------+--------------------------+------------+----------+
| hostgroup | digest             | SUBSTR(digest_text,0,25) | count_star | sum_time |
+-----------+--------------------+--------------------------+------------+----------+
| 3         | 0x9D058B6F3BC2F754 | SELECT c FROM sbtest4 WH | 250480     | 56924508 |
| 3         | 0x99396EC34E1F41D4 | SELECT c FROM sbtest8 WH | 250802     | 56425387 |
| 3         | 0x6DD78C71FF7350AE | SELECT c FROM sbtest7 WH | 249390     | 56315225 |
| 3         | 0x9B090963F41AD781 | SELECT c FROM sbtest10 W | 248580     | 56310945 |
| 3         | 0x03744DC190BC72C7 | SELECT c FROM sbtest5 WH | 248471     | 56221219 |
| 3         | 0xBF001A0C13781C1D | SELECT c FROM sbtest1 WH | 246000     | 55810714 |
| 3         | 0x0250CB4007721D69 | SELECT c FROM sbtest3 WH | 245850     | 55695979 |
| 3         | 0xD84E4E04982951C1 | SELECT c FROM sbtest9 WH | 246550     | 55670635 |
| 3         | 0x9AF59B998A3688ED | SELECT c FROM sbtest2 WH | 246370     | 55619716 |
| 3         | 0x1E7B7AC5611F30C2 | SELECT c FROM sbtest6 WH | 246710     | 55490024 |
| 3         | 0x283AA9863F85EFC8 | SELECT DISTINCT c FROM s | 25048      | 30024233 |
| 3         | 0x63F9BD89D906209B | SELECT DISTINCT c FROM s | 25080      | 30012143 |
| 3         | 0x847CD40BA8EA5175 | SELECT DISTINCT c FROM s | 24847      | 29725980 |
| 3         | 0x16BD798E66615299 | SELECT DISTINCT c FROM s | 24939      | 29691094 |
| 3         | 0x79AAA81E70A6243C | SELECT DISTINCT c FROM s | 24858      | 29682977 |
| 3         | 0x44BCB144058686EB | SELECT DISTINCT c FROM s | 24585      | 29548012 |
| 3         | 0x4AC6CC3E8E66E2A5 | SELECT DISTINCT c FROM s | 24637      | 29495328 |
| 3         | 0x578BA02C8A0C4542 | SELECT DISTINCT c FROM s | 24671      | 29446580 |
| 3         | 0x1C601BD49EB4AEB1 | SELECT DISTINCT c FROM s | 24655      | 29370584 |
| 3         | 0xC19480748AE79B4B | SELECT DISTINCT c FROM s | 24600      | 29348712 |
| 3         | 0x0D3830CC26B680E5 | SELECT c FROM sbtest4 WH | 25048      | 14392816 |
| 3         | 0x6CF96E6AF162E10B | SELECT c FROM sbtest8 WH | 25080      | 14263479 |
| 3         | 0x135C4A683DA43FBB | SELECT c FROM sbtest7 WH | 24939      | 14211630 |
| 3         | 0xEAAF7F1BA74AEA24 | SELECT c FROM sbtest10 W | 24858      | 14205801 |
| 3         | 0xB809E77ADDD9E05B | SELECT c FROM sbtest5 WH | 24847      | 14196501 |
| 3         | 0x640E62FBAA3B9666 | SELECT c FROM sbtest9 WH | 24655      | 14096192 |
| 3         | 0xAC80A5EA0101522E | SELECT c FROM sbtest1 WH | 24600      | 14072070 |
| 3         | 0x0FEDA5F1D95F2FBA | SELECT c FROM sbtest3 WH | 24585      | 14063901 |
| 3         | 0x2BD5CA9A9C3B517D | SELECT c FROM sbtest2 WH | 24637      | 14030717 |
| 3         | 0x569D075505D85717 | SELECT c FROM sbtest6 WH | 24671      | 14006483 |
| 3         | 0x4E06883FCBCBCF10 | SELECT c FROM sbtest7 WH | 24939      | 9705466  |
| 3         | 0x7A7055660FB56763 | SELECT c FROM sbtest8 WH | 25080      | 9678299  |
| 3         | 0xC13153E29C19B751 | SELECT c FROM sbtest6 WH | 24671      | 9670280  |
| 3         | 0xC2A4F66B0CA11A02 | SELECT c FROM sbtest4 WH | 25048      | 9614958  |
| 3         | 0x80D0D77C28AAE370 | SELECT c FROM sbtest5 WH | 24847      | 9569474  |
| 3         | 0x5E694374F7F03EBE | SELECT c FROM sbtest10 W | 24858      | 9530566  |
| 3         | 0x290B92FD743826DA | SELECT c FROM sbtest1 WH | 24600      | 9476476  |
| 3         | 0x381AAD21F4326865 | SELECT c FROM sbtest2 WH | 24637      | 9461251  |
| 3         | 0xD085AECB03B951B2 | SELECT c FROM sbtest9 WH | 24655      | 9461128  |
| 3         | 0x4607134A412A3661 | SELECT c FROM sbtest3 WH | 24585      | 9442733  |
| 3         | 0x4438008ADF85B3AE | SELECT SUM(K) FROM sbtes | 25048      | 8769026  |
| 3         | 0x5D10B1F28B54F53C | SELECT SUM(K) FROM sbtes | 24847      | 8743382  |
| 3         | 0xFCF5F7499618B983 | SELECT SUM(K) FROM sbtes | 25080      | 8705125  |
| 3         | 0xC23CB2971AEFA309 | SELECT SUM(K) FROM sbtes | 24939      | 8667797  |
| 3         | 0x9BC3442A424EA0DC | SELECT SUM(K) FROM sbtes | 24637      | 8666106  |
| 3         | 0x4D3EFF9C797A029D | SELECT SUM(K) FROM sbtes | 24655      | 8650351  |
| 3         | 0xA5C4BB53B8C119FD | SELECT SUM(K) FROM sbtes | 24858      | 8639550  |
| 3         | 0x3138A29396F2C746 | SELECT SUM(K) FROM sbtes | 24585      | 8627039  |
| 3         | 0x8B9F0559BA064E1C | SELECT SUM(K) FROM sbtes | 24600      | 8542927  |
| 3         | 0xA9416FB6528F2E29 | SELECT SUM(K) FROM sbtes | 24671      | 8524866  |
| 1         | 0x99396EC34E1F41D4 | SELECT c FROM sbtest8 WH | 40720      | 8136353  |
| 1         | 0x03744DC190BC72C7 | SELECT c FROM sbtest5 WH | 40758      | 8098528  |
| 1         | 0xD84E4E04982951C1 | SELECT c FROM sbtest9 WH | 40380      | 8060072  |
| 1         | 0x9AF59B998A3688ED | SELECT c FROM sbtest2 WH | 39730      | 8020855  |
| 1         | 0x9D058B6F3BC2F754 | SELECT c FROM sbtest4 WH | 39918      | 7928638  |
| 1         | 0x0250CB4007721D69 | SELECT c FROM sbtest3 WH | 39953      | 7924841  |
| 1         | 0x1E7B7AC5611F30C2 | SELECT c FROM sbtest6 WH | 39927      | 7912619  |
| 1         | 0xBF001A0C13781C1D | SELECT c FROM sbtest1 WH | 39771      | 7901737  |
| 1         | 0x6DD78C71FF7350AE | SELECT c FROM sbtest7 WH | 40235      | 7898421  |
| 1         | 0x9B090963F41AD781 | SELECT c FROM sbtest10 W | 39540      | 7880114  |
| 1         | 0x63F9BD89D906209B | SELECT DISTINCT c FROM s | 4071       | 4903017  |
| 1         | 0x44BCB144058686EB | SELECT DISTINCT c FROM s | 3995       | 4840433  |
| 1         | 0x847CD40BA8EA5175 | SELECT DISTINCT c FROM s | 4074       | 4836955  |
| 1         | 0x16BD798E66615299 | SELECT DISTINCT c FROM s | 4023       | 4836834  |
| 1         | 0xC19480748AE79B4B | SELECT DISTINCT c FROM s | 3976       | 4834656  |
| 1         | 0x1C601BD49EB4AEB1 | SELECT DISTINCT c FROM s | 4038       | 4812447  |
| 1         | 0x283AA9863F85EFC8 | SELECT DISTINCT c FROM s | 3990       | 4754678  |
| 1         | 0x578BA02C8A0C4542 | SELECT DISTINCT c FROM s | 3992       | 4754202  |
| 1         | 0x79AAA81E70A6243C | SELECT DISTINCT c FROM s | 3954       | 4678037  |
| 1         | 0x4AC6CC3E8E66E2A5 | SELECT DISTINCT c FROM s | 3973       | 4667418  |
| 1         | 0xB809E77ADDD9E05B | SELECT c FROM sbtest5 WH | 4075       | 2258591  |
| 1         | 0x6CF96E6AF162E10B | SELECT c FROM sbtest8 WH | 4071       | 2232384  |
| 1         | 0x640E62FBAA3B9666 | SELECT c FROM sbtest9 WH | 4038       | 2229571  |
| 1         | 0x569D075505D85717 | SELECT c FROM sbtest6 WH | 3992       | 2225877  |
| 1         | 0x135C4A683DA43FBB | SELECT c FROM sbtest7 WH | 4023       | 2215570  |
| 1         | 0x0FEDA5F1D95F2FBA | SELECT c FROM sbtest3 WH | 3995       | 2212948  |
| 1         | 0xAC80A5EA0101522E | SELECT c FROM sbtest1 WH | 3977       | 2195711  |
| 1         | 0x0D3830CC26B680E5 | SELECT c FROM sbtest4 WH | 3990       | 2176985  |
| 1         | 0xEAAF7F1BA74AEA24 | SELECT c FROM sbtest10 W | 3954       | 2170099  |
| 1         | 0x2BD5CA9A9C3B517D | SELECT c FROM sbtest2 WH | 3973       | 2161105  |
| 1         | 0x7A7055660FB56763 | SELECT c FROM sbtest8 WH | 4072       | 1493439  |
| 1         | 0x80D0D77C28AAE370 | SELECT c FROM sbtest5 WH | 4075       | 1479836  |
| 1         | 0xD085AECB03B951B2 | SELECT c FROM sbtest9 WH | 4038       | 1475852  |
| 1         | 0xC13153E29C19B751 | SELECT c FROM sbtest6 WH | 3992       | 1464915  |
| 1         | 0x5E694374F7F03EBE | SELECT c FROM sbtest10 W | 3954       | 1456455  |
| 1         | 0x4E06883FCBCBCF10 | SELECT c FROM sbtest7 WH | 4023       | 1449762  |
| 1         | 0x4607134A412A3661 | SELECT c FROM sbtest3 WH | 3995       | 1446750  |
| 1         | 0xC2A4F66B0CA11A02 | SELECT c FROM sbtest4 WH | 3990       | 1443391  |
| 1         | 0x381AAD21F4326865 | SELECT c FROM sbtest2 WH | 3973       | 1443232  |
| 1         | 0x290B92FD743826DA | SELECT c FROM sbtest1 WH | 3977       | 1434898  |
| 1         | 0x4D3EFF9C797A029D | SELECT SUM(K) FROM sbtes | 4038       | 1310735  |
| 1         | 0xFCF5F7499618B983 | SELECT SUM(K) FROM sbtes | 4072       | 1301788  |
| 1         | 0x5D10B1F28B54F53C | SELECT SUM(K) FROM sbtes | 4075       | 1299801  |
| 1         | 0x3138A29396F2C746 | SELECT SUM(K) FROM sbtes | 3995       | 1291833  |
| 1         | 0xC23CB2971AEFA309 | SELECT SUM(K) FROM sbtes | 4023       | 1286426  |
| 1         | 0xA5C4BB53B8C119FD | SELECT SUM(K) FROM sbtes | 3954       | 1282266  |
| 1         | 0x4438008ADF85B3AE | SELECT SUM(K) FROM sbtes | 3990       | 1279605  |
| 1         | 0x8B9F0559BA064E1C | SELECT SUM(K) FROM sbtes | 3977       | 1277712  |
| 1         | 0x9BC3442A424EA0DC | SELECT SUM(K) FROM sbtes | 3973       | 1276952  |
| 1         | 0xA9416FB6528F2E29 | SELECT SUM(K) FROM sbtes | 3992       | 1258573  |
+-----------+--------------------+--------------------------+------------+----------+
100 rows in set (0.01 sec)
```
## 十四、其他配置
### 14.1 线程的ping_interval_time
由于在后台mysql的server端设置了连接超时，对于sleep的线程，超时后将会被kill

在数据流量小的情况下，会有连接被kill，但是proxysql在再次请求mysql的时候并不知道上次的连接已经断开，

直接使用上次的连接进行操作，将会出现问题。
```sql 
set mysql-ping_interval_server_msec=10000;
set mysql-monitor_groupreplication_healthcheck_interval = 1000;
load mysql variables to runtime;
save mysql variables to disk;
```
更改proxysql中上述参数

每秒查询一次sys.gr_member_routing_candidate_status以获取MGR中节点状态

每10秒钟进行一次连接，保证proxysql到mysql的连接存在

更改完成后，不再报错

### 14.2 更改proxysql连接到MySQL_server的线程数
```sql
set mysql-threads = 64;
SAVE MYSQL VARIABLES TO DISK;
PROXYSQL RESTART
```
该配置需要重启后生效

### 14.3 proxysql提供MySQL服务的端口更改
执行如下命令
```sql
set mysql-interfaces=  0.0.0.0:3306;
SAVE MYSQL VARIABLES TO DISK;
PROXYSQL RESTART
```
更改该端口也需要重启proxysql，配置才能生效
