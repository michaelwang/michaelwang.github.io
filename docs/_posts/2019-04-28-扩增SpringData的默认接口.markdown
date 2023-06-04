---
layout: post
title:  "扩展SpringData的默认接口"
date:   2019-04-28 11:01:11 +0900
categories: java
---



## 概述

最近由于工作关系，需要对Spring Data的默认暴露的HTTP接口进行改造，以适应公司现有的系统通信协议规范。

至于spring data是个什么框架，可以[参考这里](https://spring.io/projects/spring-data)。 spring data默认会为一个实体类比如User，暴露标准的Restful风格标准的接口，比如创建用户：

    $ curl -i -X POST -H "Content-Type:application/json" -d '{  "name" : "Test", \ 
    "email" : "test@test.com" }' http://localhost:8080/users
    {
      "name" : "test",
      "email" : "test@test.com",
      "_links" : {
        "self" : {
          "href" : "http://localhost:8080/users/1"
        },
        "websiteUser" : {
          "href" : "http://localhost:8080/users/1"
        }
      }
    }
    
查询用户：

    curl -X POST -H "Content-Type:application/json"  http://localhost:8080/users/1
    {
      "name" : "test",
      "email" : "test@test.com",
      "_links" : {
        "self" : {
          "href" : "http://localhost:8080/users/1"
        },
        "websiteUser" : {
          "href" : "http://localhost:8080/users/1"
        }
      }
    }
    
而我需要做的事情就是在spring data默认暴露的restful 接口基础上再扩展一套符合公司目前使用的http接口协议：

### 创建用户
    curl -X POST -H 'Content-Type:Application/json' http://localhost:8080/users/create
    {
     "name" : "test", 
     "website: "www.test.com"
    }
    
### 查询用户
    curl -X GET -H "Content-Type:Application/json" http://locahost:8080/users/select
    {
    "namePatter" : "test"
    }
     
## 思考和探索的过程
 
1. 首先Spring Data默认的 http接口是从Dispatch Servlet暴露出来的

2. Dispatch Servlet会被servlet容器启动，这个启动的入口类会扫描class path下的jar包，并完成IOC容器的构建。

3. 在IOC容器构建的过程中，会扫描被@Controller标注的class信息，并把这些信息注册到Handler Mapping中，Mapping中保存了在Controller中定义的URL和处理类的映射关系。

4. 当请求http请求发送Spring的时候， http请求中的参数会是通过一系列的Argument Resolver对象，每个Resolver对象将request中的参数会解析成一个具体的强类型的对象，比如Date类型的对象，或者Pageable类型的对象。 

5. 当前请求被分发到Dispatch Servlet后，Spring会从Handler Mapping中寻找和当前URL匹配的Controller Handler，该Handler中其实就是目标方法，Spring会通过反射的形式，并结合步骤4中解析出来的强类型参数，执行目标方法。

如上的步骤就是Dispatch Servlet的执行过程，但是Spring Data是如何将接口暴露出来的？ 

带着这个问题阅读了下spring data源码，发现它的处理逻辑如下：

1.  Spring Data会首先声明一个Template Controller类，这个类中包含了有restful风格的接口方法定义。

2. Spring Data会再声明一个标注，@RepositoryController，该标注会被使用在步骤1的模版类上。 

3. Dispatch Servlet会基于步骤2中的标注加载步骤1的Template Controller，并将模版中的方法和URL 注入到全局的Handler Mapping对象中去。 

4. 在步骤3中的Template Controller持有一个Repositories对象，该对象就是所有的实体类的Repositoy对象的集合至于该对象是如何产生的，和本文主题不相关，先不去深入研究。

5. 当有请求发送过来，于是就会有优先根据Template Controller中注册到的Handler Mapping对象中的规则进行匹配，然后再根据Domain Class匹配到相应的repository，然后执行该repository中的方法。 

 

分析到如上步骤，我就有了如下的思路，可以仿造Spring Data的方式，再扩展一套模版Controller方法，在该Controller中按照现有的通信协议写对应的Controller方法，然后注入Entity Manager对象，然后利用JPA的Criteria 对象动态构建查询对象进行查询。 

这个事情前后持续了大概有4-5个星期。后面我会整理一下，把相关的代码贴到GitHub上，供大家参考学习。