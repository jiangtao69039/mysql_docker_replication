# 使用docker搭建mysql集群(Mysql Group Replication)  

搭建环境经测试在debian系列系统可用,centos系列未测试请自己尝试  
机器防火墙已关闭

#### 拉取镜像
```shell
docker pull mysql:8.0.20 
```  

#### 创建docker网络
```shell
docker network create --subnet=172.72.0.0/24 mysql-network
```  

#### 创建目录存储数据
```shell
sudo mkdir -p /usr/local/mysql
sudo chmod -R 777 /usr/local/mysql
mkdir -p /usr/local/mysql/lhrmgr15/conf.d
mkdir -p /usr/local/mysql/lhrmgr15/data
mkdir -p /usr/local/mysql/lhrmgr16/conf.d
mkdir -p /usr/local/mysql/lhrmgr16/data
mkdir -p /usr/local/mysql/lhrmgr17/conf.d
mkdir -p /usr/local/mysql/lhrmgr17/data
```

#### 创建3个mysql容器
```shell
docker run -d --name mysql8020mgr33065 \
   -h lhrmgr15 -p 33065:3306 --net=mysql-network --ip 172.72.0.15 \
   -v /usr/local/mysql/lhrmgr15/conf.d:/etc/mysql/conf.d -v /usr/local/mysql/lhrmgr15/data:/var/lib/mysql/ \
   -e MYSQL_ROOT_PASSWORD=lhr \
   -e TZ=Asia/Shanghai \
   mysql:8.0.20

docker run -d --name mysql8020mgr33066 \
   -h lhrmgr16 -p 33066:3306 --net=mysql-network --ip 172.72.0.16 \
   -v /usr/local/mysql/lhrmgr16/conf.d:/etc/mysql/conf.d -v /usr/local/mysql/lhrmgr16/data:/var/lib/mysql/ \
   -e MYSQL_ROOT_PASSWORD=lhr \
   -e TZ=Asia/Shanghai \
   mysql:8.0.20

docker run -d --name mysql8020mgr33067 \
   -h lhrmgr17 -p 33067:3306 --net=mysql-network --ip 172.72.0.17 \
   -v /usr/local/mysql/lhrmgr17/conf.d:/etc/mysql/conf.d -v /usr/local/mysql/lhrmgr17/data:/var/lib/mysql/ \
   -e MYSQL_ROOT_PASSWORD=lhr \
   -e TZ=Asia/Shanghai \
   mysql:8.0.20
```

#### 确认上面三个mysql容器启动成功并且能连上,能通过机器ip+容器映射的端口连上了才能下一步操作
```shell
docker logs -f mysql8020mgr33067  
docker exec -it mysql8020mgr33065 mysql -uroot -plhr 
```

#### 修改mysql配置文件
```shell
cat > /usr/local/mysql/lhrmgr15/conf.d/my.cnf <<"EOF"
[mysqld]
user=mysql
port=3306
character_set_server=utf8mb4
secure_file_priv=''
server-id = 802033065
default-time-zone = '+8:00'
log_timestamps = SYSTEM
log-bin = 
binlog_format=row
binlog_checksum=NONE
log-slave-updates=1
skip-name-resolve
auto-increment-increment=2
auto-increment-offset=1
gtid-mode=ON
enforce-gtid-consistency=on
default_authentication_plugin=mysql_native_password
max_allowed_packet = 500M

master_info_repository=TABLE
relay_log_info_repository=TABLE
relay_log=lhrmgr15-relay-bin-ip15


transaction_write_set_extraction=XXHASH64
loose-group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
loose-group_replication_start_on_boot=OFF
loose-group_replication_local_address= "172.72.0.15:33061"
loose-group_replication_group_seeds= "172.72.0.15:33061,172.72.0.16:33062,172.72.0.17:33063"
loose-group_replication_bootstrap_group=OFF
loose-group_replication_ip_whitelist="172.72.0.15,172.72.0.16,172.72.0.17"

report_host=172.72.0.15
report_port=3306

EOF


cat >  /usr/local/mysql/lhrmgr16/conf.d/my.cnf <<"EOF"
[mysqld]
user=mysql
port=3306
character_set_server=utf8mb4
secure_file_priv=''
server-id = 802033066
default-time-zone = '+8:00'
log_timestamps = SYSTEM
log-bin = 
binlog_format=row
binlog_checksum=NONE
log-slave-updates=1
gtid-mode=ON
enforce-gtid-consistency=ON
skip_name_resolve
default_authentication_plugin=mysql_native_password
max_allowed_packet = 500M

master_info_repository=TABLE
relay_log_info_repository=TABLE
relay_log=lhrmgr16-relay-bin-ip16


transaction_write_set_extraction=XXHASH64
loose-group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
loose-group_replication_start_on_boot=OFF
loose-group_replication_local_address= "172.72.0.16:33062"
loose-group_replication_group_seeds= "172.72.0.15:33061,172.72.0.16:33062,172.72.0.17:33063"
loose-group_replication_bootstrap_group=OFF
loose-group_replication_ip_whitelist="172.72.0.15,172.72.0.16,172.72.0.17"

report_host=172.72.0.16
report_port=3306

EOF


cat > /usr/local/mysql/lhrmgr17/conf.d/my.cnf <<"EOF"
[mysqld]
user=mysql
port=3306
character_set_server=utf8mb4
secure_file_priv=''
server-id = 802033067
default-time-zone = '+8:00'
log_timestamps = SYSTEM
log-bin = 
binlog_format=row
binlog_checksum=NONE
log-slave-updates=1
gtid-mode=ON
enforce-gtid-consistency=ON
skip_name_resolve
default_authentication_plugin=mysql_native_password
max_allowed_packet = 500M


master_info_repository=TABLE
relay_log_info_repository=TABLE
relay_log=lhrmgr16-relay-bin-ip16


transaction_write_set_extraction=XXHASH64
loose-group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
loose-group_replication_start_on_boot=OFF
loose-group_replication_local_address= "172.72.0.17:33063"
loose-group_replication_group_seeds= "172.72.0.15:33061,172.72.0.16:33062,172.72.0.17:33063"
loose-group_replication_bootstrap_group=OFF
loose-group_replication_ip_whitelist="172.72.0.15,172.72.0.16,172.72.0.17"

report_host=172.72.0.17
report_port=3306

EOF
```  

#### 重启mysql应用上面的配置
```shell
docker restart mysql8020mgr33065 mysql8020mgr33066 mysql8020mgr33067
```

#### 安装MGR插件(在3个mysql容器都需要执行)
```shell
#这是第一个mysql容器
docker exec -it mysql8020mgr33065 mysql -uroot -plhr 
INSTALL PLUGIN group_replication SONAME 'group_replication.so';
```
```shell
#这是第二个mysql容器
docker exec -it mysql8020mgr33066 mysql -uroot -plhr 
INSTALL PLUGIN group_replication SONAME 'group_replication.so';
```
```shell
#这是第三个mysql容器
docker exec -it mysql8020mgr33067 mysql -uroot -plhr 
INSTALL PLUGIN group_replication SONAME 'group_replication.so';
```

--------------------

#### 设置集群帐号(在3个mysql容器都需要执行)
```shell
#这是第一个mysql容器
docker exec -it mysql8020mgr33065 mysql -uroot -plhr 
SET SQL_LOG_BIN=0;
CREATE USER repl@'%' IDENTIFIED BY 'lhr';
GRANT REPLICATION SLAVE ON *.* TO repl@'%';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;
CHANGE MASTER TO MASTER_USER='repl', MASTER_PASSWORD='lhr' FOR CHANNEL 'group_replication_recovery';
```
```shell
#这是第二个mysql容器
docker exec -it mysql8020mgr33066 mysql -uroot -plhr 
SET SQL_LOG_BIN=0;
CREATE USER repl@'%' IDENTIFIED BY 'lhr';
GRANT REPLICATION SLAVE ON *.* TO repl@'%';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;
CHANGE MASTER TO MASTER_USER='repl', MASTER_PASSWORD='lhr' FOR CHANNEL 'group_replication_recovery';
```
```shell
#这是第三个mysql容器
docker exec -it mysql8020mgr33067 mysql -uroot -plhr 
SET SQL_LOG_BIN=0;
CREATE USER repl@'%' IDENTIFIED BY 'lhr';
GRANT REPLICATION SLAVE ON *.* TO repl@'%';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;
CHANGE MASTER TO MASTER_USER='repl', MASTER_PASSWORD='lhr' FOR CHANNEL 'group_replication_recovery';
```

-----------------------  

#### 启动MGR单主模式(仅在第一个mysql容器执行)
```shell
#这是第一个mysql容器
docker exec -it mysql8020mgr33065 mysql -uroot -plhr 
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;
#查看MGR组信息 
SELECT * FROM performance_schema.replication_group_members;
```  

#### 另外两个节点加入(在第二个,第三个mysql容器执行)
```shell
#这是第二个mysql容器
docker exec -it mysql8020mgr33066 mysql -uroot -plhr 
START GROUP_REPLICATION;
```

```shell
#这是第三个mysql容器
docker exec -it mysql8020mgr33067 mysql -uroot -plhr 
START GROUP_REPLICATION;
```

--------------

#### 查看集群状态
```shell
docker exec -it mysql8020mgr33065 mysql -uroot -plhr 
SELECT * FROM performance_schema.replication_group_members;
```

--------------

### 多主和单主模式切换
```shell
#查看当前模式
docker exec -it mysql8020mgr33065 mysql -uroot -plhr 
show variables like '%group_replication_single_primary_mode%';
SELECT @@group_replication_single_primary_mode;
```

```shell
#单主 切换 成多主
docker exec -it mysql8020mgr33065 mysql -uroot -plhr 
select group_replication_switch_to_multi_primary_mode();
#查询信息
SELECT * FROM performance_schema.replication_group_members; 
```  

```shell
# 多主切换成单主
#查询信息,查出要当主的MEMBER_ID
SELECT * FROM performance_schema.replication_group_members; 
#切换成单主,参数是 上面查出来的MEMBER_ID
select group_replication_switch_to_single_primary_mode('67090f47-d785-11ea-b76c-0242ac480010') ;
#查询信息
SELECT * FROM performance_schema.replication_group_members; 

```
