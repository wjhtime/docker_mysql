# docker实现mysql主从
linux环境下docker实现mysql主从服务器



## 思路

启动2个mysql服务器，mysql_master主服务器，mysql_slave从服务器，并实现主从复制功能

另外docker实现的nginx负载均衡在此 [nginx负载均衡](https://github.com/wjhtime/docker_mysql)



## 目录说明

```
/root/docker/mysql/conf.d/master.cnf	主机master配置文件
/etc/mysql/conf.d/master.cnf	master容器的配置文件
/root/docker/mysql/master-data	主机master数据文件
/var/lib/mysql	容器中mysql数据文件
/root/docker/mysql/conf.d/slave.cnf	主机中slave从服务器配置文件
/etc/mysql/conf.d/slave.cnf	slave容器中配置文件
/root/docker/mysql/slave-data	slave容器中的数据文件
```



## 步骤

1. 启动mysql_master主服务器

```shell
docker pull mysql:5.7

docker run --name mysql_master -d -p 3307:3306 -v /root/docker/mysql/conf.d/master.cnf:/etc/mysql/conf.d/master.cnf -v /root/docker/mysql/master-data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root mysql:5.7
```



2. 启动mysql_slave从服务器

```shell
docker run --name mysql_slave -d -p 3308:3306 -v /root/docker/mysql/conf.d/slave.cnf:/etc/mysql/conf.d/slave.cnf -v /root/docker/mysql/slave-data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root --link mysql_master:mysql_master mysql:5.7
```



3. 创建backup用户，并授权同步

```
grant replication slave on *.* to 'backup'@'%' indentified by '111111';
```



查看主服务器的状态，记录pos等数据

```
show master status;
```



4. 从服务器中执行

```
change master to master_host='mysql_master',master_user='backup',master_password='111111',master_file_log='mysql_bin.000003',master_log_pos=439,master_port=3306;
```



```mysql
# 查看各自的状态
show slave status;
show master status;

# 启动主从复制
start slave;
```



mysql_master.cnf配置

```mysql
[mysqldump]
user=root
password='root'
[mysqld]
max_allowed_packet=8M
lower_case_table_names=1
character_set_server=utf8
max_connections=900
max_connect_errors=600
# 比较重要，主服务器的id是1
server_id=1
log-bin=mysql-bin

slow_query_log=1
long_query_time=1
log_error
```



mysql_slave.cnf配置

```mysql
[mysqldump]
user=root
password='root'
[mysqld]
max_allowed_packet=8M
lower_case_table_names=1
character_set_server=utf8
max_connections=900
max_connect_errors=600
# 比较重要，从服务器的id是2
server_id=2
log-bin=mysql-bin

slow_query_log=1
long_query_time=1
log_error
```



两个的log-bin配置均采用mysql-bin的方式



5. 后续增加从服务器

```mysql
# master服务器
flush tables with read lock;

# 记录日志点
show master status;

# 导出
mysql_dump -R uroot -p database > /tmp/database.sql
unlock tables;

从服务器的配置方法同上
创建一个server_id=3的配置文件，一个数据挂载目录
source /tmp/database.sql
change master to ...
```









