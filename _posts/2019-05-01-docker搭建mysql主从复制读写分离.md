---
layout:     post
title:      docker搭建mysql主从复制读写分离
date:       2019-05-01
author:     cyf
header-img: img/beautiful.jpg
catalog: true
tags:
    - Mysql
    - Docker
---
# 一、主从库搭建过程
```
docker run -d -p 3316:3306 -v /home/cyf/docker/mysql/master/config/:/etc/mysql -v /home/cyf/docker/mysql/master/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 --net cyf --ip 172.18.0.2 --name master 1347445564/mysql5.7.18last
docker run -d -p 3326:3306 -v /home/cyf/docker/mysql/slaver/config/:/etc/mysql -v /home/cyf/docker/mysql/slaver/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 --net cyf --ip 172.18.0.3 --name slaver 1347445564/mysql5.7.18last
```
# 二、搭建主从复制

主库配置文件
```
[mysqld]
必要的配置：
log-bin=master-mysql-bin //开启二进制
binlog-format=MIXED //mysql日志格式（Row、Statement、Mixed)
server_id=1 //与从库不能相同，唯一

可选的配置：
binlog-ignore-db = mysql //不同步的数据库（mysql通常不同步）
binlog-do-db = db_name //同步的数据库
expire_logs_days = 10 //binlog日志保留的天数
```
[mysql日志格式](https://blog.csdn.net/mycwq/article/details/17136997)

从库配置文件
```
[mysqld]
server_id=2
```
主库操作
```
mysql> grant replication slave on *.* to root@'172.18.0.3' identified by '123456';
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> show master status;
+-------------------------+----------+--------------+------------------+-------------------+
| File                    | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+-------------------------+----------+--------------+------------------+-------------------+
| master-mysql-bin.000003 |      446 |              |                  |                   |
+-------------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```
从库操作
```
mysql> change master to master_user='root',master_password='123456',master_host='172.18.0.2',master_log_file='master-mysql-bin.000003',master_log_pos=446;
Query OK, 0 rows affected, 2 warnings (0.08 sec)

mysql> start slave;
Query OK, 0 rows affected (0.00 sec)
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.18.0.2
                  Master_User: root
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master-mysql-bin.000003
          Read_Master_Log_Pos: 446
               Relay_Log_File: f78474f07f6e-relay-bin.000002
                Relay_Log_Pos: 327
        Relay_Master_Log_File: master-mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```
# 三、搭建读写分离

360atlas搭建
```
docker run -d -p 1234:1234 -p 2345:2345 -v /home/cyf/docker/mysql/atlas/conf/:/usr/local/mysql-proxy/conf -v /home/cyf/docker/mysql/atlas/logs:/usr/local/mysql-proxy/log --name 360atlas --net cyf --ip 172.18.0.4 registry.cn-beijing.aliyuncs.com/qianjia2018/qianjia_dev:atlas
```
配置文件
```
[mysql-proxy]

#管理接口的用户名
admin-username=root

#管理接口的密码
admin-password=123456

#实现管理接口的Lua脚本所在路径
admin-lua-script=/usr/local/mysql-proxy/lib/mysql-proxy/lua/admin.lua

#Atlas后端连接的MySQL主库的IP和端口，可设置多项，用逗号分隔
proxy-backend-addresses=172.18.0.1:3316

#Atlas后端连接的MySQL从库的IP和端口，@后面的数字代表权重，用来作负载均衡，若省略则默认为1，可设置多项，用逗号分隔
proxy-read-only-backend-addresses=172.18.0.1:3326

#设置Atlas的运行方式，设为true时为守护进程方式，设为false时为前台方式，一般开发调试时设为false，线上运行时设为true
daemon=false

#设置Atlas的运行方式，设为true时Atlas会启动两个进程，一个为monitor，一个为worker，monitor在worker意外退出后会自动将其重启，设为false时只有worker，没有monitor，一般开发调试时设为false，线上运行时设为true
keepalive=true

#工作线程数，推荐设置与系统的CPU核数相等
event-threads=4

#日志级别，分为message、warning、critical、error、debug五个级别
log-level=message

#日志存放的路径
log-path=/usr/local/mysql-proxy/log

#实例名称，用于同一台机器上多个Atlas实例间的区分
instance=test

#Atlas监听的工作接口IP和端口
proxy-address=0.0.0.0:1234

#Atlas监听的管理接口IP和端口
admin-address=0.0.0.0:2345

#连接池的最小空闲连接数，应设为event-threads的整数倍，可根据业务请求量大小适当调大或调小
min-idle-connections=8

#分表设置，此例中person为库名，mt为表名，id为分表字段，3为子表数量，可设置多项，以逗号分隔，若不分表则不需要设置该项
#tables = person.mt.id.3

#用户名与其对应的加密过的MySQL密码，密码使用PREFIX/bin目录下的加密程序encrypt加密，此设置项用于多个用户名同时访问同一个Atlas实例的情况，若只有一个用户名则不需要设置该项
#pwds = user1:+jKsgB3YAG8=, user2:GS+tr4TPgqc=

#默认字符集，若不设置该项，则默认字符集为latin1
charset=utf8

#允许连接Atlas的客户端的IP，可以是精确IP，也可以是IP段，以逗号分隔，若不设置该项则允许所有IP连接，否则只允许列表中的IP连接
#client-ips = 127.0.0.1, 192.168.1

#Atlas前面挂接的LVS的物理网卡的IP(注意不是虚IP)，若有LVS且设置了client-ips则此项必须设置，否则可以不设置
#lvs-ips = 192.168.1.1

pwds=root:/iZxz+0GRoA=
```
360atlas操作
```
mysql -h127.0.0.1 -P2345 -uroot -p123456 //登录360atlas管理端口
mysql> select * from help;
+----------------------------+---------------------------------------------------------+
| command                    | description                                             |
+----------------------------+---------------------------------------------------------+
| SELECT * FROM help         | shows this help                                         |
| SELECT * FROM backends     | lists the backends and their state                      |
| SET OFFLINE $backend_id    | offline backend server, $backend_id is backend_ndx's id |
| SET ONLINE $backend_id     | online backend server, ...                              |
| ADD MASTER $backend        | example: "add master 127.0.0.1:3306", ...               |
| ADD SLAVE $backend         | example: "add slave 127.0.0.1:3306", ...                |
| REMOVE BACKEND $backend_id | example: "remove backend 1", ...                        |
| SELECT * FROM clients      | lists the clients                                       |
| ADD CLIENT $client         | example: "add client 192.168.1.2", ...                  |
| REMOVE CLIENT $client      | example: "remove client 192.168.1.2", ...               |
| SELECT * FROM pwds         | lists the pwds                                          |
| ADD PWD $pwd               | example: "add pwd user:raw_password", ...               |
| ADD ENPWD $pwd             | example: "add enpwd user:encrypted_password", ...       |
| REMOVE PWD $pwd            | example: "remove pwd user", ...                         |
| SAVE CONFIG                | save the backends to config file                        |
| SELECT VERSION             | display the version of Atlas                            |
+----------------------------+---------------------------------------------------------+
16 rows in set (0.00 sec)

mysql> select * from backends;
+-------------+-----------------+-------+------+
| backend_ndx | address         | state | type |
+-------------+-----------------+-------+------+
|           1 | 172.18.0.1:3316 | up    | rw   |
|           2 | 172.18.0.1:3326 | up    | ro   | //up状态即可
+-------------+-----------------+-------+------+
2 rows in set (0.00 sec)
```
# 四、360插件大坑---数据乱码
主要原因是连接360atlas服务用的账号密码不一致

**问题出现时的情况**

java服务1的配置文件

```
  datasource:
      driver-class-name: com.mysql.jdbc.Driver
      url: jdbc:mysql://${ip}:1234/test1?useSSL=false&verifyServerCertificate=false&useUnicode=true&characterEncoding=UTF8
      username: dbadmin_app
      password: ###########
```

java服务2的配置文件

```
  datasource:
      driver-class-name: com.mysql.jdbc.Driver
      url: jdbc:mysql://${ip}:1234/test2?useSSL=false&verifyServerCertificate=false&useUnicode=true&characterEncoding=UTF8
      username: root
      password: **********
```
两个服务连接的360atlas代理端口用的用户密码不同，导致从代理端口写入的数据是乱码的。

**解决方案**

```
统一连接360atlas代理端口所用的账号密码即可

删除了360atlas配置文件test.cnf的root用户和密码

- pwds=root:+BKmpgcePdna8dbmijJjEw==,dbadmin_app:WpbeJoiOzod7IdiSIpk2Rw==
+ pwds=dbadmin_app:WpbeJoiOzod7IdiSIpk2Rw==
重启360atlas即可（重启注意事项：将连接360atlas代理端口的服务停止，重启360atlas时，再重新启动）

```
具体详情看[软件架构-mysql主从](https://www.jishuwen.com/d/2dCZ)