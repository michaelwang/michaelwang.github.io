---
layout: post
title:  "SpringBoot-Async-线程池爆满问题解决过程"
date:   2018-12-07 11:01:11 +0900
categories: java
---

## 问题概述

前几天生产环境的购物车服务突然不能正常访问，分析日志中的报错信息，发现是购物车使用的异步工作线程池的队列达到了设置的上限， 致使新的http请求生成的task不能被继续放入到线程池的队列中。这是直接的报错原因，深入分析日志，发现有大量的超时错误，这些超时错误 都是发生在购物车服务调用另一个用户中心服务时发生的。所以问题是由于大量的超时请求将工作线程池中的线程消耗光导致。 故障发生持续了10-15分钟，最后是以重启购物车服务的方式恢复服务。

下面就基于该问题进行详细的分析过程。

## 超时错误分析

首先查看问题发生时的错误日志信息，在20:00 至 20:05 之间，从日志系统统计下来有大概1600 到 1700条超时的错误日志

      2018-11-28 20:05:00 [qtp2073069810-113] ERROR c.s.s.d.b.a.v.ApiShopCartV3Controller - ???????:80353735-4a4c-44f1-ab8b-b45e34123d49
      java.util.concurrent.ExecutionException: ???3??????
          at com.shlf.shopcart.data.kernel.misc.AsyncUtils.getAsyncResult(AsyncUtils.java:20) ~[classes!/:0.0.1-SNAPSHOT]
          at com.shlf.shopcart.data.kernel.service.ShopCartService.doGetShoppingCart(ShopCartService.java:346) ~[classes!/:0.0.1-SNAPSHOT]
          at com.shlf.shopcart.data.kernel.service.ShopCartService$$FastClassBySpringCGLIB$$34b21853.invoke() ~[classes!/:0.0.1-SNAPSHOT]
          at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:204) ~[spring-core-4.3.14.RELEASE.jar!/:4.3.14.RELEASE]
          at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:669) ~[spring-aop-4.3.14.RELEASE.jar!/:4.3.14.RELEASE]
          at com.shlf.shopcart.data.kernel.service.ShopCartService$$EnhancerBySpringCGLIB$$1db1c5b9.doGetShoppingCart() ~[classes!/:0.0.1-SNAPSHOT]
          at com.shlf.shopcart.data.biz.api.v3.ApiShopCartV3Controller.loadCartRequest(ApiShopCartV3Controller.java:403) [classes!/:0.0.1-SNAPSHOT]
          at sun.reflect.GeneratedMethodAccessor631.invoke(Unknown Source) ~[?:?]
          at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_171]
          at java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_171]
          ......
          at java.lang.Thread.run(Thread.java:748) [?:1.8.0_171]
      Caused by: java.util.concurrent.TimeoutException
          at java.util.concurrent.FutureTask.get(FutureTask.java:205) ~[?:1.8.0_171]
          at com.shlf.shopcart.data.kernel.misc.AsyncUtils.getAsyncResult(AsyncUtils.java:17) ~[classes!/:0.0.1-SNAPSHOT]
          ... 75 more
       
从这个错误堆栈，我们可以得到如下的信息
 1. 这是由于发起远程服务调用查询时产生的超时错误
 2. Controller层接收到请求之后，业务代码使用了异步模式来处理逻辑
 3. 这个方法:com.**.AsyncUtils.getAsyncResult的地方抛出了TimeoutException

## 线程池拒绝执行错误分析

在20:05到20:10之间，大量出现如下错误，统计下来大概有10656条记录

     2018-11-28 20:10:00 [qtp2073069810-17182] ERROR c.s.s.d.b.a.v.ApiShopCartV3Controller - ???????:56ec6a2c-b431-45ee-bd8c-287742df4f7f
     org.springframework.core.task.TaskRejectedException: Executor [java.util.concurrent.ThreadPoolExecutor@a792059[Running, pool size = 500, active threads = 500, queued tasks = 5000, completed tasks = 13363050]] did not accept task: org.springframework.aop.interceptor.AsyncExecutionInterceptor$1@6518d14d
         at org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor.submit(ThreadPoolTaskExecutor.java:323) ~[spring-context-4.3.14.RELEASE.jar!/:4.3.14.RELEASE]
         at org.springframework.aop.interceptor.AsyncExecutionAspectSupport.doSubmit(AsyncExecutionAspectSupport.java:277) ~[spring-aop-4.3.14.RELEASE.jar!/:4.3.14.RELEASE]
         at org.springframework.aop.interceptor.AsyncExecutionInterceptor.invoke(AsyncExecutionInterceptor.java:130) ~[spring-aop-4.3.14.RELEASE.jar!/:4.3.14.RELEASE]
         at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179) ~[spring-aop-4.3.14.RELEASE.jar!/:4.3.14.RELEASE]
         at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:673) ~[spring-aop-4.3.14.RELEASE.jar!/:4.3.14.RELEASE]
     Caused by: java.util.concurrent.RejectedExecutionException: Task java.util.concurrent.FutureTask@7684d81a rejected from java.util.concurrent.ThreadPoolExecutor@a792059[Running, pool size = 500, active threads = 500, queued tasks = 5000, completed tasks = 13363050]
         at java.util.concurrent.ThreadPoolExecutor$AbortPolicy.rejectedExecution(ThreadPoolExecutor.java:2063) ~[?:1.8.0_171]
         at java.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:830) ~[?:1.8.0_171]
         at java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1379) ~[?:1.8.0_171]
         at java.util.concurrent.AbstractExecutorService.submit(AbstractExecutorService.java:134) ~[?:1.8.0_171]
         at org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor.submit(ThreadPoolTaskExecutor.java:320) ~[spring-context-4.3.14.RELEASE.jar!/:4.3.14.RELEASE]
         ... 80 more
     
代码分析

购物车的代码使用了spring boot的async的模式来组织代码，当请求进入到controller中后，调用service对象的方法，service中的方法是被async标注的方法。 当某一个service方法又调用其他service对象中的被async标注的方法的时后，后续执行的方法是否会使用线程池的其他线程来执行还是使用 当前的线程来执行？基于这个疑问，我搭建了一个工程进行验证，验证的伪代码如下：

    @Async
    public Future<String> hello(){
         System.out.println(Thread.currentThread().getName());
         AsyncResult r = new AsyncResult("hello");
         return r;
    }
    
    
    @Async
    public Future<String> world(){
       System.out.println(Thread.currentThread().getName());
       Future<String> hello = hello();
       String helloString = hello.get();
       AsyncResult r = new AsyncResult(helloString + " world");
       return r;
    }
    
    
    @SrpingBootApplication
    @EnableAsync
    public static void main(String args[]){
         System.out.println(world().get);
    }
    
    
    @Configuration
    public class AsyncConfig{
    
    
       @Bean
       public Executor taskExecutor(){
          ......
          executor.setCoreSize(2);
          executor.setMaxSize(2);
          executor.setQueueSize(2);
    
    
          ......
       }
    
    
    }
    
运行如上的代码，代码的第3、11行分别输出如下信息：

    ShhiangAsyncExecutor-1
    ShihangAsyncExecutor-2

说明在Spring Boot的Async编程模式中，如果有多个异步方法被顺序调用，后续执行的每一个异步的方法都会使用不同的线程执行，而不是复用前一个方法占用的线程。 那么问题就来了，如果有一个异步方法执行的是远程过程调用，而这个远程过程调用的超时时间是30秒，而如果此时恰巧这个远程服务 出现了性能下降，或者网络抖动，比如出现了丢包现象，那么这个线程就会一直阻塞在这次远程过程调用中，直到30秒之后，该线程才能够被 释放出来，开始服务下一个请求，那么线程池中的线程的工作效率依赖这个远程过程调用的返回时间的快慢，如果远程调用的响应越快， 线程池的效率就越高，如果远程调用响应越慢，那么线程池的效率就越低。

那么我们再回到这个问题，根据上述的异常信息和服务恢复的方式来看，可以推测这是是由于远程过程调用中有大量的超时请求， 这些超时请求耗光了线程池中的所有线程，又耗光了线程池中的所有的队列空间。

## 问题总结

这个问题的临时快速的解决方案可以将不同的服务用不同的线程池进行处理，避免共用同一个线程池而引起的交叉感染。 长远的更合理的方式，应该在服务调用之间加入网关，利用限流、融断机制，当某一个服务出现故障的时后，将有问题的 服务尽快从请求链中隔离开，避免引起滚雪球的效应，对其他服务产生影响，本问题其时就是购物车依赖的用户中心服务， 出现了故障，这个出了故障的外部服务耗光了购物车线程池中的所有资源，间接导致了购物车不能继续正常服务。所以在 系统集成的时后，我们需要尤其关注这些集成点之间逻辑。