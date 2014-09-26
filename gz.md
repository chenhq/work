TSM故障分析报告
==============


故障分析处理过程
--------------


15点28分，客户端使用异常，排查客户端和平台测的日志发现连接Oracle异常，Oracle的连接数已经超过最大值。

```
java.sql.SQLException: ORA-00018: maximum number of sessions exceeded

        at oracle.jdbc.driver.DatabaseError.throwSqlException(DatabaseError.java:112)
        at oracle.jdbc.driver.T4CTTIoer.processError(T4CTTIoer.java:331)
        at oracle.jdbc.driver.T4CTTIoer.processError(T4CTTIoer.java:283)
        at oracle.jdbc.driver.T4CTTIoer.processError(T4CTTIoer.java:278)
        at oracle.jdbc.driver.T4CTTIoauthenticate.receiveOsesskey(T4CTTIoauthenticate.java:288)
        at oracle.jdbc.driver.T4CConnection.logon(T4CConnection.java:357)
        at oracle.jdbc.driver.PhysicalConnection.<init>(PhysicalConnection.java:414)
        at oracle.jdbc.driver.T4CConnection.<init>(T4CConnection.java:165)
        at oracle.jdbc.driver.T4CDriverExtension.getConnection(T4CDriverExtension.java:35)
        at oracle.jdbc.driver.OracleDriver.connect(OracleDriver.java:801)
        at com.alibaba.druid.filter.FilterChainImpl.connection_connect(FilterChainImpl.java:142)
        at com.alibaba.druid.filter.FilterAdapter.connection_connect(FilterAdapter.java:768)
        at com.alibaba.druid.filter.FilterEventAdapter.connection_connect(FilterEventAdapter.java:38)
        at com.alibaba.druid.filter.FilterChainImpl.connection_connect(FilterChainImpl.java:136)
        at com.alibaba.druid.filter.stat.StatFilter.connection_connect(StatFilter.java:211)
        at com.alibaba.druid.filter.FilterChainImpl.connection_connect(FilterChainImpl.java:136)
        at com.alibaba.druid.pool.DruidAbstractDataSource.createPhysicalConnection(DruidAbstractDataSource.java:1280)
        at com.alibaba.druid.pool.DruidAbstractDataSource.createPhysicalConnection(DruidAbstractDataSource.java:1334)
        at com.alibaba.druid.pool.DruidDataSource.init(DruidDataSource.java:420)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:39)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
        at java.lang.reflect.Method.invoke(Method.java:597)
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.invokeCustomInitMethod(AbstractAutowireCapableBeanFactory.java:1581)
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.invokeInitMethods(AbstractAutowireCapableBeanFactory.java:1522)
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.initializeBean(AbstractAutowireCapableBeanFactory.java:1452)
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:519)
        at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:456)
        at org.springframework.beans.factory.support.AbstractBeanFactory$1.getObject(AbstractBeanFactory.java:294)
        at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:225)
        at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:291)
        at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:193)
```

验证连接数据库，发现长时间无法连接：

```
[cttsm@push1 ~]$ !telnet
telnet 10.235.156.170 1521
Trying 10.235.156.170...
```

登录数据库服务器，发现长时间无法登录Oracle：

```
bash-3.00$ sqlplus /nolog

SQL*Plus: Release 10.2.0.1.0 - Production on Fri Sep 26 16:09:03 2014

Copyright (c) 1982, 2005, Oracle.  All rights reserved.

SQL> connect / as sysdba
Connected.
SQL> SQL> SQL> SQL> SQL> SQL> SQL> SQL> SQL> SQL> SQL> SQL> SQL>
```

检查数据库的资源发现CPU和内存都已经用尽，Load等待队列超过35，说明CPU资源严重不足，同时超高的内核空间CPU使用，预示着磁盘的压力非常大。

```
load averages:  35.01,  40.63,  46.37;                    up 646+17:55:56
336 processes: 331 sleeping, 1 stopped, 4 on cpu
CPU states: 0.0% idle, 36.2% user,  63.8% kernel,  0.0% iowait,  0.0% swap
Memory: 8064M phys mem,   6M free mem, 16G total swap, 12G free swap

   PID USERNAME LWP PRI NICE  SIZE   RES STATE    TIME    CPU COMMAND
 24187 oracle    11  50    0 2684M 2514M sleep   65:47  3.56% oracle
 24129 oracle    11  11    0 2684M 2514M sleep   65:13  2.54% oracle
 24121 oracle    11   0    0 2684M 2514M sleep   64:58  1.51% oracle
 24106 oracle    11  50    0 2684M 2514M sleep   64:41  1.46% oracle
 15155 oracle    11  21    0 2684M 2516M cpu/16  19:17  1.42% oracle
 15494 oracle    11   0    0 2684M 2516M cpu/24  19:08  1.41% oracle
 24155 oracle    11  10    0 2685M 2515M cpu/8   67:14  1.25% oracle
 25412 patrol     1  59    0   16M   10M sleep   40.8H  1.24% dcm
 24176 oracle    11  59    0 2684M 2514M sleep   67:53  1.23% oracle
 28773 patrol     1  47    2 2168K 1928K sleep    0:00  1.16% sadc
 22569 oracle     3  50    0   39M   22M sleep  558.7H  1.14% tnslsnr
 24098 oracle    11  59    0 2684M 2514M sleep   67:10  1.11% oracle
 28818 oracle     1  50    0 2681M 2515M sleep    0:00  1.10% oracle
 28816 oracle     1  50    0 2681M 2515M sleep    0:00  1.10% oracle
 28814 oracle     1  50    0 2681M 2515M sleep    0:00  1.09% oracle
```

检查数据库的连接，发现数据库的连接数超过450：

```
bash-3.00$ netstat -an | grep 1521 | wc -l
     465
```

检查Oracle alert日志发现，大量session超限的告警，同时伴随着大量的连接超时的错误：

```
Fri Sep 26 14:56:02 2014
Errors in file /data/admin/dxota/bdump/dxota_smon_21189.trc:
ORA-00604: error occurred at recursive SQL level 1
ORA-00018: maximum number of sessions exceeded
Fri Sep 26 14:56:12 2014
Errors in file /data/admin/dxota/bdump/dxota_smon_21189.trc:
ORA-00604: error occurred at recursive SQL level 1
ORA-00018: maximum number of sessions exceeded
Fri Sep 26 14:56:22 2014
Errors in file /data/admin/dxota/bdump/dxota_smon_21189.trc:
ORA-00604: error occurred at recursive SQL level 1
ORA-00018: maximum number of sessions exceeded
Fri Sep 26 14:56:32 2014
Errors in file /data/admin/dxota/bdump/dxota_smon_21189.trc:
ORA-00604: error occurred at recursive SQL level 1
ORA-00018: maximum number of sessions exceeded
Fri Sep 26 14:56:42 2014
Errors in file /data/admin/dxota/bdump/dxota_smon_21189.trc:
ORA-00604: error occurred at recursive SQL level 1
ORA-00018: maximum number of sessions exceeded
Fri Sep 26 14:56:52 2014
Errors in file /data/admin/dxota/bdump/dxota_smon_21189.trc:
ORA-00604: error occurred at recursive SQL level 1
ORA-00018: maximum number of sessions exceeded
Fri Sep 26 14:57:02 2014
Errors in file /data/admin/dxota/bdump/dxota_smon_21189.trc:
ORA-00604: error occurred at recursive SQL level 1
ORA-00018: maximum number of sessions exceeded
Fri Sep 26 14:57:13 2014
```

```
WARNING: inbound connection timed out (ORA-3136)
Fri Sep 26 15:05:55 2014
WARNING: inbound connection timed out (ORA-3136)
Fri Sep 26 15:05:55 2014
WARNING: inbound connection timed out (ORA-3136)
Fri Sep 26 15:05:55 2014
WARNING: inbound connection timed out (ORA-3136)
Fri Sep 26 15:05:55 2014
WARNING: inbound connection timed out (ORA-3136)
Fri Sep 26 15:05:55 2014
WARNING: inbound connection timed out (ORA-3136)
Fri Sep 26 15:05:55 2014
WARNING: inbound connection timed out (ORA-3136)
Fri Sep 26 15:05:55 2014
WARNING: inbound connection timed out (ORA-3136)
Fri Sep 26 15:05:55 2014
WARNING: inbound connection timed out (ORA-3136)
Fri Sep 26 15:05:55 2014
WARNING: inbound connection timed out (ORA-3136)
```


此时数据库已无法通过sqlplus管理，通过讨论确定，关闭部分服务器的应用，为数据库减轻压力。

关闭push1上面的应用和OTA的部分应用，数据库链接下降：


```
bash-3.00$ netstat -an | grep 1521 | wc -l
     38
```


重启启动暂停的应用后，连接数稳定在145左右。

```
bash-3.00$ netstat -an | grep 1521 | wc -l
     145
```

检查应用发现运行正常，检查数据库，数据压力降低，测试业务流程均正常。


故障结论
-------

应用压力增加，导致数据库的压力过大，加上数据库本身的性能非常低下，导致应用执行的sql语句无法执行完成，连接池的连接无法释放和重用，最终累积数据库阻塞和连接数超过最大值。
