---
layout: post
title:  "Kubernetes的学习总结"
date:   2020-02-16 11:01:11 +0900
categories: 技术
---
## 概述

最近陆陆续续的看了一些Kubernetes的资料，这篇文章对最近的学习做一个总结性的输出，本文主要介绍

1.Kubernetes的设计哲学

2.各个构件的作用介绍

3.创建一个Pod的流程介绍

阅读完本文大概你会了解如上的内容，大概需要1-3分钟阅读完本文。

## Kubernetes的设计哲学

Kubernetes是的前身是Google基于它的Brog论文 研发的内部的容器编排工具，据说google的员工需要签署保密协议，不能向外透露这个系统的相关信息，直到2015年左右，google才开源这个系统。所以从这里看出google对于这个系统的重视程度之高。

我们在日常的运维中遇到的最大难点就是服务的扩容和升级。因为要考虑很多方面的问题包括机柜、服务器的配置、负载均衡的设置、同时多版本在线的流量切换等等，每个方面的问题都需要花费很多的精力才能做好。Kubernetes主要就是解决这些方面的问题。

Kubernetes是主要是基于Master-Slave的模式设计，其中Master负责核心逻辑控制，包括扩容集群的计算节点、Pod调度、状态控制。Slave主要是向Master节点汇报相关信息包括：Pod的运行情况、Node节点的剩余计算资源等信息。Master节点负责维护集群的状态，如果当前集群的状态不是期望的状态，Master节点会不断修正状态到期望的状态。

## 各个构建的作用介绍

### 1、Api Server

这个是运行在Master节点上，所有的Pod、Node、Service、Replication Controller、End Point的创建都是通过Api Server创建的，提供的List，Watch等接口，可以让其他构建监听资源的变化；

### 2、Kubectl

这是运行在Master节点上的，是Api Server的一个CLI工具，具备的1中Api Server所有的能力；

### 3、Kube Schedule

定时调度组件，运行在Master节点上，所有的负责监听Api Server中的资源变化，并负责实际的资源创建和状态更新。

### 4、Etcd

Kubernetes的所有的资源的状态信息和资源信息都存在Etcd中，所有的etcd的写入都是通过Api Server写入的，这个组建也是运行在Master节点上的。

### 5、Kubelet

这是运行在Node节点上的组建，负责管理Node节点的信息，上报Node的信息到Api Server，然后由Api Server将Node的信息写入到Etcd中。

### 6、Kube-Proxy

这个组件既可以以运行在Master节点上，也可以运行在Node节点上，可以代理Api Server，也可以代理node上运行的Pod服务。

## 创建一个Pod的流程介绍

1、客户端通过Kubectl创建一个Pod；

2、Kubectl负责把信息通过Api Server写入到Etcd；

3、Kube Schedule 监听到Pod创建的请求负责创建Pod，然后从现有的Node中选取满足条件的Node信息；

4、Kublet 监听到Api Server中的创建Pod的信息后，负责拉去pod的描述信息，并拉去容器的镜像信息，然后在node中创建Pod，并调用Api Server将信息写入到Etcd中。

## 总结
本文介绍了kubernetes的架构设计和哲学，一些核心的组件的功能还有这些组件在Pod创建过程中是如何配合的，
在后面的系列文章中会详细介绍每个构件详细工作原理。