---
layout: post
title:  "Hibernate中的实体状态切换"
date:   2019-06-02 11:01:11 +0900
categories: java
---

# 概述

我们在使用ORM工具时，直接和实体进行交互，就可以操作数据库，不再需要写 SQL语句。但是Hibernate不是软件饮弹，我们还是需要关注它生成出来SQL。那么在一次操作的过程中实体会经历哪些状态？各个状态又有哪些含义和作用？

# 详细解释
## Transient

一个对象刚创建时的状态，这个状态下的对象还没有映射为数据库中的记录。只有调用Session.save()或者 EntityManager.persist()方法的时候，状态会切换为Managed。

## Managed

刚创建的对象通过调用Session.save()或者EntityManager.persist()方法的时候，就会从 Transient状态切换到Managed状态，该状态下的对象的任何属性值改变都会在Session.flush()的时候，自动同步到数据库中，因此我们不需要手动执行insert/update/delete这些方法。

## Detatched
如果 Session或者EntityManager关闭了，那么原来是在 Session或者entityManager管理范围内的对象，就会处于Detatched状态。在该状态下，对象的后续的任何改变都不会同步到数据库中，但是可以通过Session.update()或者EntityManager.merge()中的方法重新返回到Managed状态。

## Removed

当调用EntityManager.remove()方法的时候，对象就会处于该状态。我们也可以通过调用EntityManager.persist()方法将对象重新返回到Managed状态。当处于Removed状态的对象，在flush()阶段的时候，会自动同步到数据库。

状态的切换，具体可以参考下图：

![hibernate-entity-state!](/assets/img/spring-data-mvc-hibernate-entity-state-machine.png "hibernate-entity-state")
