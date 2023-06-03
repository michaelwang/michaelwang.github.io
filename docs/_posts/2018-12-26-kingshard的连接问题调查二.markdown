---
layout: post
title:  "kingshard的连接问题调查(二)"
date:   2018-12-12 11:01:11 +0900
categories: java
---

继续之前的分析 kingshard的连接问题调查(一) 中的推测，是由于JDBC的connection pool中的链接失效导致发生如下错误

    communications link failure the last packet sent successfully to the server was 0 milliseconds ago

为了验证这个推测，在本地模拟了一个测试场景，具体如下：

1. 将Mysql Server的超时时间调整为30秒
2. 将应用中JDBC连接池(Druid)连接最大闲置检查时间间隔设置为60秒
3. 应用启动后，等待30秒之后再去访问一个有数据库查询的Http接口

当过了30秒之后，Mysql服务端就有如下的日志信息：

    2018-12-26T03:11:00.156070Z 300 [Note] Aborted connection 300 to db: 'usercenter' user: 'usercenter' host: '10.1.44.162' (Got timeout reading communication packets)
    2018-12-26T03:11:00.156070Z 299 [Note] Aborted connection 299 to db: 'usercenter' user: 'usercenter' host: '10.1.44.162' (Got timeout reading communication packets)
    2018-12-26T03:11:00.161399Z 301 [Note] Aborted connection 301 to db: 'usercenter' user: 'usercenter' host: '10.1.44.162' (Got timeout reading communication packets)
    2018-12-26T03:11:01.492482Z 303 [Note] Aborted connection 303 to db: 'usercenter' user: 'usercenter' host: '10.1.44.162' (Got timeout reading communication packets)
    
可以看出是Mysql Server主动关闭了应用发送过来的连接。
在30秒之后，再在系统层面的检查Java应用的链接状态如下：

    [michael@localhost fresh-shopcart]$ lsof -i -a -p 9721
    COMMAND  PID    USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
    java    9721 michael   45u  IPv6 27850531      0t0  TCP localhost.localdomain:47604->10.68.251.213:webcache (ESTABLISHED)
    java    9721 michael   49u  IPv6 27853075      0t0  TCP localhost.localdomain:45180->10.1.62.105:mysql (CLOSE_WAIT)
    java    9721 michael   50u  IPv6 27850549      0t0  TCP localhost.localdomain:45182->10.1.62.105:mysql (CLOSE_WAIT)
    java    9721 michael   51u  IPv6 27853980      0t0  TCP localhost.localdomain:45184->10.1.62.105:mysql (CLOSE_WAIT)
    java    9721 michael   52u  IPv6 27850550      0t0  TCP localhost.localdomain:45186->10.1.62.105:mysql (CLOSE_WAIT)
    java    9721 michael   53u  IPv6 27853981      0t0  TCP localhost.localdomain:45188->10.1.62.105:mysql (CLOSE_WAIT)
    
可以看出连接都是处于close_wait状态，关于close_wait状态的含义和作用可以参考我的另一个文章：TCP链接状态分析

Java应用层面的报错信息如下：

    com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure
     
    The last packet successfully received from the server was 44,120 milliseconds ago.  The last packet sent successfully to the server was 1 milliseconds ago.
        at sun.reflect.GeneratedConstructorAccessor123.newInstance(Unknown Source)
        at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
        at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
        at com.mysql.jdbc.Util.handleNewInstance(Util.java:404)
        at com.mysql.jdbc.SQLError.createCommunicationsException(SQLError.java:988)
        at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:3552)
        at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:3452)
        at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:3893)
        at com.mysql.jdbc.MysqlIO.sendCommand(MysqlIO.java:2526)
        at com.mysql.jdbc.MysqlIO.sqlQueryDirect(MysqlIO.java:2673)
        at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2545)
    
可以看出Java的应用在执行数据库操作时发生了异常。

综合这些错误信息证明了之前的推测是对的， Mysql Server服务端主动关闭链接后， 而Java应用的数据库连接池需要等待一个时间窗口才能感知到连接池里的连接已经失效了， 在本例中时间窗口就是30秒(60-30=30)，在这个时间窗口内如果再有请求发送过来， Druid就会分配一个实际上已经失效的连接给应用使用，于是就出现 Communications link failure 错误。

## 结论和下一步计划

**数据库的超时检查时间一定要大于应用的超时检查时间**

如上是在没有加入kingshard中间件下的验证，下面就要加入kingshard进行验证：Druid有定时的闲置连接检测的机制，这个机制发出的指令被kingshard接受到之后，没有转发到mysql的服务端，导致kingshard和mysql之间的连接由于达到了mysql的超时设置而被关闭，但是kignshard并没有最后关闭这些连接，这些连接一直处于close_wait状态（为什么是处于close_wait状态，参考我的另一篇文章 TCP链接状态分析），于是就引起了连接泄漏的问题。