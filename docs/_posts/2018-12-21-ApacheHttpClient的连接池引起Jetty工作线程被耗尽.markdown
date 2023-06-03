---
layout: post
title:  "ApacheHttpClient的连接池引起Jetty工作线程被耗尽"
date:   2018-12-21 11:01:11 +0900
categories: java
---

## 概述

12月20日下午2点到4点左右，商城商品的商品评论不能被加载出来，当时生产环境有3个商品服务的实例，根据日志报错信息来看，有大量的超时

报错信息，超时是由于调用用户中心的服务接口导致的，再排查用户中心的服务的状态，发现用户中心的3个服务的网络流量比平时都大了10倍，

为了尽快恢复服务，将用户中心的服务再平行扩展出一个节点，然后再将商品评论的3个服务中的两个重启，保留一个服务作为问题调查的现场。然后线上

服务恢复正常。经过一番分析，定位出这是由于商品评论服务调用用户中心服务时出现了大量的http错误，导致fresh-rpc组件中关于Apache HttpClient对象中的connection不能被释放回归到connection pool中，而新的http请求又需要从 connection pool 中获取connection，如果没有获取到connection就返回一个错误，也算一个合理的处理方案，但是http client的作者认为从 connection pool 中获取 connection肯定会成功，所以作者使用了很理想主义的代码来获取connection，这就导致了所有的jetty的工作线程都阻塞在连接池对象的get方法上，从而导致了商品评论服务不能正常对外提供服务。

下面是问题排查过程的详细描述。

 
## 排查过程

为了能够排查出问题的原因，以及验证解决方案是否正确，我们首先需要将故障现场进行重现，重现故障现象的步骤如下：

1. 将商品评论服务在本地启动 
2. 部署usercenter data服务到  10.1.62.47 服务器上 
3. 因为商品评论服务是通过zk发现usercenter data服务的，所以修改dev环境中的 zk上的地址为步骤2中的地址
4. 将10.1.62.47上的丢包概率提升到50%-70%之间   
5. 本地压力测试商品评论服务接口

从商品评论服务的超时错误的日志（如下），以及用户中心的服务器的负载情况来看，推测是由于远程服务调用发生了超时，导致jetty的线程池中的线程不能回归到可被复用的状态。 

    Caused by: com.shlf.rpc.transport.TransportException: java.net.SocketTimeoutException: Read timed out
        at com.shlf.rpc.transport.defaults.ApacheHttpClientTransport.flush(ApacheHttpClientTransport.java:201) ~[fresh-rpc-core-1.1-GA.jar!/:?]
        at com.shlf.rpc.binding.MapperMethod.transfer(MapperMethod.java:220) ~[fresh-rpc-core-1.1-GA.jar!/:?]
        at com.shlf.rpc.binding.MapperMethod.convertResult(MapperMethod.java:91) ~[fresh-rpc-core-1.1-GA.jar!/:?]
        at com.shlf.rpc.binding.MapperMethod.execute(MapperMethod.java:58) ~[fresh-rpc-core-1.1-GA.jar!/:?]
        at com.shlf.rpc.binding.MapperProxy.invoke(MapperProxy.java:50) ~[fresh-rpc-core-1.1-GA.jar!/:?]
        ... 70 more
    Caused by: java.net.SocketTimeoutException: Read timed out
        at java.net.SocketInputStream.socketRead0(Native Method) ~[?:1.8.0_171]
        at java.net.SocketInputStream.socketRead(SocketInputStream.java:116) ~[?:1.8.0_171]
        at java.net.SocketInputStream.read(SocketInputStream.java:171) ~[?:1.8.0_171]
        at java.net.SocketInputStream.read(SocketInputStream.java:141) ~[?:1.8.0_171]
        at org.apache.http.impl.io.SessionInputBufferImpl.streamRead(SessionInputBufferImpl.java:137) ~[httpcore-4.4.6.jar!/:4.4.6]
       

为了验证这个推测，在本地开发环境基于重现步骤进行验证，发现即使usercenter data服务的接口有超时发生，jetty的线程也会快速回归到可被复用的状态。

所以说推测由于超时而引起jetty工作线程被耗尽是不正确的。

商品服务调用 用户中心没有设置超时，而是采用的默认超时配置30秒，所以推测是否是在30秒的时间窗口内，积压了超过jetty工作线程数的请求数量，而引起服务不可用。 

基于这个假设，如果usercenter data 服务恢复正常，请求量降下来，商品服务应该也可以恢复正常。但是到保留下来的商品评论服务器上用如下命令检查，发现服务还是处于不可用状态。 

    curl http://localhost:9070

但是检查发现9070的端口还是存在的。所以30秒积压了超过jetty工作线程数量的请求导致服务不可用的推测也不成立。但是这里又引起思考了另一个问题：java进程还活着，但是为什么http的接口没有任何响应？所以猜测是不是jetty工作线程都处于死锁状态下。

通过排查线程栈，发现有211个线程都是在等待连接，具体如下：

    java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for <0x0000000088092468> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
        at org.apache.http.pool.AbstractConnPool.getPoolEntryBlocking(AbstractConnPool.java:377)
        at org.apache.http.pool.AbstractConnPool.access$200(AbstractConnPool.java:67)
        at org.apache.http.pool.AbstractConnPool$2.get(AbstractConnPool.java:243)
        - locked <0x0000000089cbd090> (a org.apache.http.pool.AbstractConnPool$2)
        at org.apache.http.pool.AbstractConnPool$2.get(AbstractConnPool.java:191)
        at org.apache.http.impl.conn.PoolingHttpClientConnectionManager.leaseConnection(PoolingHttpClientConnectionManager.java:303)
        at org.apache.http.impl.conn.PoolingHttpClientConnectionManager$1.get(PoolingHttpClientConnectionManager.java:279)
        at org.apache.http.impl.execchain.MainClientExec.execute(MainClientExec.java:191)
        at org.apache.http.impl.execchain.ProtocolExec.execute(ProtocolExec.java:185)
        at org.apache.http.impl.execchain.RetryExec.execute(RetryExec.java:89)
        at org.apache.http.impl.execchain.RedirectExec.execute(RedirectExec.java:110)
        at org.apache.http.impl.client.InternalHttpClient.doExecute(InternalHttpClient.java:185)
        at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:118)
        at org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:56)
        at com.shlf.rpc.transport.defaults.ApacheHttpClientTransport.flush(ApacheHttpClientTransport.java:182)
        at com.shlf.rpc.binding.MapperMethod.transfer(MapperMethod.java:220)
        at com.shlf.rpc.binding.MapperMethod.convertResult(MapperMethod.java:91)
        at com.shlf.rpc.binding.MapperMethod.execute(MapperMethod.java:58)
        at com.shlf.rpc.binding.MapperProxy.invoke(MapperProxy.java:50)
        at com.sun.proxy.$Proxy125.select(Unknown Source)
        
    
推测可能是由于调用usercenter data的连接未能释放导致所有的jetty工作线程都在等待释放的连接导致的问题。 

商品评论是java服务，usercenter data是dot net 服务，java调用dot net 是通过内部封装的一个rpc组件进行的，

根据上面的线程堆栈中的如下信息

    at org.apache.http.pool.AbstractConnPool$2.get(AbstractConnPool.java:243)
    - locked <0x0000000089cbd090> (a org.apache.http.pool.AbstractConnPool$2)

继续猜测，是否是jetty的工作线程从 Apache HttpClient 的Connection Pool中获取connection对象时被阻塞。
因此再检查rpc组件中关于发起远程调用代码逻辑，其中有如下的代码：

    try {
        httpPost = new HttpPost(getUrl().getPath());
        httpPost.setConfig(RequestConfig.copy(RequestConfig.DEFAULT).setConnectTimeout(connectTimeout).setSocketTimeout(readTimeout).build());
        if (!CollectionUtils.isEmpty(customHeaders)) {
            for (Map.Entry<String, String> header : customHeaders.entrySet()) {
                httpPost.setHeader(header.getKey(), header.getValue());
            }
        }
        httpPost.setEntity(new ByteArrayEntity(data));
        HttpResponse response = httpClient.execute(httpHost, httpPost);
        int responseCode = response.getStatusLine().getStatusCode();
        if (responseCode != HttpStatus.SC_OK) {
            throw new TransportException("HTTP response code :" + responseCode);
        }
        is = response.getEntity().getContent();
        byte[] buf = new byte[2048];
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        int len = 0;
        do {
            len = is.read(buf);
            if (len > 0) {
                baos.write(buf, 0, len);
            }
        } while (-1 != len);
        EntityUtils.consumeQuietly(response.getEntity());
        inputStream = new ByteArrayInputStream(baos.toByteArray());
        IOUtils.closeQuietly(baos);
    } catch (Exception ex) {
        throw new TransportException(ex);
    }
        
其中第12行，预期判断usercenter data肯定会返回200的响应，这是基于之前Dot Net给的协议规范： 接口都以200返回，通过在响应体的中的code体现成功还是失败的信息，作出的一个假设。

这里的嫌疑比较大，因为如果usercenter data返回的不是200，这里的逻辑那么就会导致 connection不会被归位到 connection pool中，因为在 第12行的逻辑直接抛出一个异常，

而没有将connection对象进行close处理，所以连接不会被归位到connection pool中。我们再检查发生故障时商品评论服务的日志，发现有如下调用usercenter data的非200响应：

    239945         ... 67 more
    239946 Caused by: com.shlf.rpc.transport.TransportException: HTTP response code :502
    239947         at com.shlf.rpc.transport.defaults.ApacheHttpClientTransport.flush(ApacheHttpClientTransport.       java:185) ~[fresh-rpc-core-1.1-GA.jar!/:?]
    239948         at com.shlf.rpc.binding.MapperMethod.transfer(MapperMethod.java:220) ~[fresh-rpc-core-1.1-GA.       jar!/:?]
    239949         at com.shlf.rpc.binding.MapperMethod.convertResult(MapperMethod.java:91) ~[fresh-rpc-core-1.1       -GA.jar!/:?]
    239950         at com.shlf.rpc.binding.MapperMethod.execute(MapperMethod.java:58) ~[fresh-rpc-core-1.1-GA.ja       r!/:?]
    239951         at com.shlf.rpc.binding.MapperProxy.invoke(MapperProxy.java:50) ~[fresh-rpc-core-1.1-GA.jar!/       :?]
    239952         ... 67 more
    
     

统计下来这个错误数为32，正好是rpc组件中设置的 connection pool 的大小

    $ cat api-mall-product-comment.log | grep 'Caused by: com.shlf.rpc.transport.TransportException: HTTP response code :502' | wc -l
      32

 

那么这里我们需要反问：为什么会出现了非200的响应？ 

检查了usercenter data服务的IIS日志，没有发现有非200的响应发出，那么这里唯一可能的解释是：

这个非200的响应不是IIS给出的，而是Windows Server给出的，而这个是我们预期之外的行为。

再继续排查rpc组件，发现在产生非200响应的时候抛出了TransportException，并没有释放repsonse.getEntity()。

经过debug后发现，responseEntity需要关闭才能够释放持有的连接，才能够将connection 释放到 connection pool中。

排查Apache HttpClient 对象的代码（如下），其中使用了 synchronized同步关键字，这个导致jetty的工作线程阻塞等待这个对象锁，

而connection pool中的connection由于usercenter data的服务返回非200的响应，池中的32个可用连接都被耗尽，但是Apache HttpClient

的作者又很确定从pool中获取连接是肯定没有问题的，所以用了一个死循环来处理获取connection对象的逻辑，于是在这么多的巧合下发生了这个故障。

 

    ......
    ......
        synchronized (this) {
    ......
                for (;;) {
                    final E leasedEntry = getPoolEntryBlocking(route, state, timeout, tunit, this);
                    if (validateAfterInactivity > 0)  {
                        if (leasedEntry.getUpdated() + validateAfterInactivity <= System.currentTimeMillis()) {
                            if (!validate(leasedEntry)) {
                                leasedEntry.close();
                                release(leasedEntry, false);
                                continue;
                            }
                        }
                    }
                    entry = leasedEntry;
                    done = true;
                    onLease(entry);
                    if (callback != null) {
                        callback.completed(entry);
                    }
                    return entry;
                }
        ......
    ......
    ......
     
 
## 解决方案和总结

1. 更新fresh-rpc 版本，对上述的非200响应进行异常处理，凡是需要释放资源的逻辑都需要写到finally块中；
2. 我们的接口层面的协议制定需要根据主流的规范标准制定而不能基于主观偏好来制定，因为主流的标准肯定是大部分中间件、服务器、操作系统都遵循的，
3. 基于大家都遵循的标准制定我们的业务实施标准、架构标准、接口规范、就会减少很多不必要的麻烦；
4. 在Windows Server层面返回的非200响应，我们还需要继续调查，是在什么前提条件下会出发这个行为？ 
