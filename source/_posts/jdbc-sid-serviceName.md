---
layout: post
title: "JDBC无法连接Oracle问题解决"
date: 2016-11-14 12:46
tags: [Oracle]
category: 工作日志
description: 使用PL/SQL可以正常访问数据库，但是在程序里使用jdbc无法获取连接。
---

## 无法连接Oracle
今天现场工程反应数据库从Oracle11g升级到12c后程序无法连接数据库，一直报错：
>Caused by: oracle.net.ns.NetException: Listener refused the connection with the following error: ORA-12505, TNS:listener does not currently know of SID given in connect descriptor 	at oracle.net.ns.NSProtocol.connect(NSProtocol.java:395) 	at oracle.jdbc.driver.T4CConnection.connect(T4CConnection.java:1102) 	at oracle.jdbc.driver.T4CConnection.logon(T4CConnection.java:320) 	... 107 more

显然是监听有问题，服务器无法识别jdbc传递的sid。下边是我的配置文件：
```
#db
db.driverClassName=oracle.jdbc.driver.OracleDriver
db.username=GEOSYS
db.password=GEOSYS
db.url=jdbc:oracle:thin:@192.168.199.109:1521:orcl
db.maxTotal=500
db.maxIdle=30
```

## 问题排查
查看网络是否连通，是否是驱动不兼容或者会
### 网络问题
关闭防火墙，问题依然存在，接着使用ssh探测端口状态：
`ssh -v -p  1521 root@192.168.199.109  `
结果网络是通的。
### plsql
使用plsql可以访问数据库，难道是sid配置有问题？
### 查看数据库实例
`select * from gv$instance;`
发现存在多个实例，于是我怀疑配置里根本就不是sid而是一个服务名。果不其然，工程根本不知道sid和服务名的区别。但是估计应该是给的服务名，于是改jdbc的连接方式。

## 更换驱动
项目使用的ojdbc6的驱动包，而且运行环境是JDK8，于是换成最新的ojdbc7。最后还是出现无法解析SID的异常。

## 问题解决
jdbc连接的两种方式：
> Oracle JDBC Thin using a ServiceName: 
jdbc:oracle:thin:@//host:port/service_name


>Oracle JDBC Thin using an SID: 
jdbc:oracle:thin:@host:port:SID

>Note: 
Support for SID is being phased out. Oracle recommends that users switch over to usingservice names. 

而且确认了现场数据库有三个实例，所以马上采用服务名的连接方式:
`db.url=jdbc:oracle:thin:@//192.168.199.109:1521/orcl`
重启服务一切正常。

## 总结
一段官方的解释最合适不过。
>What is the difference between Oracle SIDs and Oracle SERVICE NAMES. One config tool looks for SERVICE NAME and then the next looks for SIDs! What's going on?!

>Oracle SID is the unique name that uniquely identifies your instance/database where as Service name is the TNS alias that you give when you remotely connect to your database and this Service name is recorded in Tnsnames.ora file on your clients and it can be the same as SID and you can also give it any other name you want.

>SERVICE_NAME is the new feature from oracle 8i onwards in which database can register itself with listener. If database is registered with listener in this way then you can use SERVICE_NAME parameter in tnsnames.ora otherwise - use SID in tnsnames.ora.

>Also if you have OPS (RAC) you will have different SERVICE_NAME for each instance.

>SERVICE_NAMES specifies one or more names for the database service to which this instance connects. You can specify multiple services names in order to distinguish among different uses of the same database. For example:
SERVICE_NAMES = sales.acme.com, widgetsales.acme.com

>You can also use service names to identify a single service that is available from two different databases through the use of replication.

>In an Oracle Parallel Server environment, you must set this parameter for every instance.
`


 