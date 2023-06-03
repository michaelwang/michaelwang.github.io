---
layout: post
title:  "kingshard的连接问题调查一"
date:   2018-12-22 11:01:11 +0900
categories: java
---

kingshard是一个基于mysql读写分离的中间件，公司目前按照目前的体量和发展速度， 急需要一个能够支持读写分离的中间件支撑业务的发展需要，在各种背景下，我们目前选择了 kingshard作为首先验证的一个中间件。 为了验证kingshard的稳定性，先在内网的环境部署kingshard服务， 然后让项目使用kingshard进行验证，内网的开发环境、测试环境都是使用频次比较高的环境，打算在这两个环境中验证1-2个星期看稳定性如何。在这个过程中频繁出现如下的 错误：

    communications link failure the last packet sent 
    successfully to the server was 0 milliseconds ago

问题发生后，先尝试重启java服务，但是在启动过程中出现不能获取到jdbc connection 的错误，所以又尝试重启kingshard服务，再重启java服务，这样java服务才能恢复正常。 由于这个问题是偶尔发生的，所以不能够很快的确定故障原因。

为了解决该问题，需要能够重现该问题，所以在本地的开发环境中尝试重现该问题，在尝试重现的过程中大概确定了解决该问题的方向，这是由于java服务、kingshard服务、mysql服务、kubernetes服务对网络链接的超时管理设置不一致导致的。Java服务从jdbc pool中获取了一个connection使用，这个连接是Java应用和Kingshard服务之间的连接，不是真正的和 Mysql Server之间的连接，而Kingshard和Mysql Server之间的连接，可能是被Kubernetes的网络或者Mysql服务给关闭了，由于Mysql对链接的默认超时时间是8个小时，Java服务的JDBC连接存活检测的间隔为60秒，也就是说java服务从JDBC Pool里获取到的链接理论上都是有效的，而实际上Kingshard和Mysql服务之间的连接都已经失效了，因此导致上述的错误。这个推测还需要继续验证关闭连接的行为是kubernetes的网络引起的还是Kingshard服务本身没有连接存活检测机制引起的。
