---
layout: post
title:  "ApacheHttpClient的连接池引起Jetty工作线程被耗尽后的TCP的连接状态分析"
date:   2018-12-25 11:01:11 +0900
categories: java
---


文章ApacheHttpClient的连接池引起Jetty工作线程被耗尽 中提到的，

如果应用中没有对如下代码中：

    HttpResponse response = httpClient.execute(httpHost, httpPost);

response 对象执行如下方法，释放链接：

    EntityUtils.consumeQuietly(response.getEntity());

就会导致connection不能回归到connection pool中。

今天对这些不能回归的connection从系统层面进行分析，

看看是处于什么状态。

模拟多次接口调用后，Jetty的工作线程都阻塞在 connectionPool中的get方法调用上：

    org.apache.http.pool.AbstractConnPool$2.get(AbstractConnPool.java:243)   - locked 
    org.apache.http.pool.AbstractConnPool$2@207c4712org.apache.http.pool.AbstractConnPool$2.get(AbstractConnPool.java:191)

然后再执行如下命令

    lsof -i -a -p $java_pid

得到如下信息：

    java    6408 michael  381u  IPv6 11136490      0t0  TCP localhost:46768->localhost:webcache (CLOSE_WAIT)
    java    6408 michael  382u  IPv6 11136494      0t0  TCP localhost:46770->localhost:webcache (CLOSE_WAIT)
    java    6408 michael  383u  IPv6 11135941      0t0  TCP localhost:46790->localhost:webcache (CLOSE_WAIT)
    java    6408 michael  384u  IPv6 11134910      0t0  TCP localhost:46792->localhost:webcache (CLOSE_WAIT)
    java    6408 michael  385u  IPv6 11137709      0t0  TCP localhost:46802->localhost:webcache (CLOSE_WAIT)
    java    6408 michael  386u  IPv6 11135974      0t0  TCP localhost:46804->localhost:webcache (CLOSE_WAIT)
    java    6408 michael  387u  IPv6 11135986      0t0  TCP localhost:46814->localhost:webcache (CLOSE_WAIT)
    java    6408 michael  388u  IPv6 11135999      0t0  TCP localhost:46816->localhost:webcache (CLOSE_WAIT)
    java    6408 michael  389u  IPv6 11137740      0t0  TCP localhost:46830->localhost:webcache (CLOSE_WAIT)
    
可以看到链接都是处于close_wait状态。

查了一些资料得知close_wait状态是通信双方中的服务端主动关闭了链接，

于是请求发起方的操作系统收到了FIN指令，

然后在系统层面将链接状态调整为close_wait状态，

接着请求发起方的应用程需要再执行关闭动作，然后链接才能释放。

如果请求方的应用没有执行关闭动作，那么connection会一直处于close_wait状态，

直到应用程序重启，这个close_wait状态的链接才能释放。

所以从这里看出，如果应用没有对链接资源进行及时的资源释放，

就会导致链接泄漏，

随着时间发展，应用的可用链接资源就越来越少，

当链接资源都被消耗光后，就会出现无可用链接故障。

而这个问题又和我现在 解决的另一个问题： kingshard的连接问题调查(一) 有一定的相关性，

怀疑是kingshard中有链接泄漏，

从而导致应用出现找不到jdbc链接，

这个假设还需要进一步进行验证。