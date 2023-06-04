---
layout: post
title:  "如何在Hibernate中配置MySQL主键自动生成策略"
date:   2019-05-17 11:01:11 +0900
categories: java
---

# 概述

我们在使用Hibernate操作MySQL数据库的时候，会遇到主键配置的问题，因为MySQL没有提供 像Oracle或者PostgreSQL的sequence机制。因此我们如果使用基于Sequence的机制生成主键时，Hibernate会自动为我们生成一个Hibernate_Sequence的表，所以当我们要创建实体的时候，Hibernate会首先从这个表里获取当前值，然后将累加后的值作为新对象的ID值，保存到数据库中。同时 MySQL本身提供主键自增长机制，我们可以结合Hibernate主键自动生成机制实现在保存新对象的时候，自动插入新的ID值，不需要在创建对象的时候明确指定ID。下面这两种的方式的详细介绍和差异对比分析。
利用MySQL的主键自增长机制创建对象

# 详细描述

## 步骤1，在创建 MySQL表的主键的时候，需要确保主键是自增长的，可以按照如下的方式设置： 

    CREATE TABLE demo ( 
    `id` bigint(20) NOT NULL PRIMARY KEY AUTO_INCREMENT, 
    `name` varchar(255) DEFAULT NULL
    )
    
## 步骤2，在Hibernate的实体的主键用如下的方式定义：

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @GenericGenerator(name = "native", strategy = "native")
    private long id;

 

基于如上的方式定义表和实体，我们用如下代码创建对象：

    Configuration configuration = new Configuration();
    factory = configuration.configure().buildSessionFactory();
    School school = new School();
    school.setName("demo");
    Session session = factory.openSession();
    Transaction tx = session.beginTransaction();
    session.persist(school);
    tx.commit();
    session.close();
    
就会生成如下格式的SQL

    insert into School (name) values (?)

如上就实现了利用MySQL的自增长机制创建对象。
 
# 利用Hibernate的Sequence机制创建对象

## 详细描述

我们首先在创建表的时候，不再需要像上面那样创建一个自增长类型的主键，而是通过hibernate_sequence表来计算出当前插入对象的最新的ID。

### 步骤1，创建sequence表和业务表

    create table School (id bigint not null, name varchar(255), primary key (id)) engine=InnoDB
    create table hibernate_sequence (next_val bigint) engine=InnoDB

### 步骤2，定义ID属性

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private long id;

注意：这里的GenerationType类型是Sequence，不再和上面的类型一样。

### 步骤3，通过如下的方式创建对象

    Configuration configuration = new Configuration();
    factory = configuration.configure().buildSessionFactory();
    School school = new School();
    school.setName("demo");
    Session session = factory.openSession();
    Transaction tx = session.beginTransaction();
    session.persist(school);
    tx.commit();
    session.close();
    
可以观察到有如下的SQL生成：

    Hibernate: insert into hibernate_sequence values ( 1 )
    Hibernate: select next_val as id_val from hibernate_sequence for update
    Hibernate: update hibernate_sequence set next_val= ? where next_val=?
    Hibernate: insert into School (name, id) values (?, ?)
    
从这里的日志可以看出，ID是先从sequence表中生成的，然后赋值到School对象中，再生成插入的SQL。

 
# 总结

我们可以利用MySQL的主键自增长机制生成ID值，也可以利用Sequence的机制生成ID值;
我们如果使用Sequence的方式，就会有性能问题，因为所有表的插入动作，都依赖hibernate_sequence表，这里必然会成为一个性能瓶颈；
Sequence会比Identity多产生而外两条更新hibernate_sequence表的sql；

综合上面推荐使用Identify的方式创建对象。

