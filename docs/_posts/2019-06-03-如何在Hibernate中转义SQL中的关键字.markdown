---
layout: post
title:  "如何在Hibernate中转义SQL中的关键字"
date:   2019-06-03 11:01:11 +0900
categories: java
---

# 概述

我们在实际的项目中，可能会将一些SQL的关键字作为实体的属性名，我们可能会使用如下的数据库：MySQL 、PostgreSQL 、SqlServer，每个数据库都有各自不同的保留关键字，因此如果我们要将这些关键字作为实体属性的名称，或者数据库的表名，数据库就会报错。如何解决这个问题？我们可以用如下的两个方法。
在实体定义的时候通过 标注的方式指定

我们如果有如下的实体定义

    @Entity
    @Setter(AccessLevel.PUBLIC)
    @Getter(AccessLevel.PUBLIC)
    @EqualsAndHashCode
    @NoArgsConstructor
    @AllArgsConstructor
    public class School {
        @Id
        @GeneratedValue(strategy = GenerationType.SEQUENCE)
        private long id;
    
        private String name;
    
        private String desc;
    
    }
    
其中desc是MySQL的关键字，这里成为了我们的属性名，于是当我们启动项目，就会报错：

    六月 03, 2019 5:55:59 下午 org.hibernate.engine.jdbc.spi.SqlExceptionHelper logExceptions
    WARN: SQL Error: 1064, SQLState: 42000
    六月 03, 2019 5:55:59 下午 org.hibernate.engine.jdbc.spi.SqlExceptionHelper logExceptions
    
    ERROR: You have an error in your SQL syntax; check the manual that corresponds to your MySQL
     server version for the right syntax to use near 'desc, name, id) values (null, 'demo', 1)' at line 1
    
从这里的错误描述来看，可以看出desc是MySQL的关键字，所以在插入数据的时候，Hibernate就会报错。

我们可以在@Column的方式来解决这个问题，具体定义如下：

    @Entity
    @Setter(AccessLevel.PUBLIC)
    @Getter(AccessLevel.PUBLIC)
    @EqualsAndHashCode
    @NoArgsConstructor
    @AllArgsConstructor
    public class School {
        @Id
        @GeneratedValue(strategy = GenerationType.SEQUENCE)
        private long id;
    
        private String name;
    
        @Column(name = "[desc]")
        private String desc;
    
    }
    
项目启动运行正常，日志输出如下。

    六月 03, 2019 6:00:30 下午 org.hibernate.resource.transaction.backend.jdbc.internal.
    DdlTransactionIsolatorNonJtaImpl getIsolatedConnection
    INFO: HHH10001501: Connection obtained from JdbcConnectionAccess 
    
    Hibernate: create table School (id bigint not null, `desc` varchar(255), name varchar(255), 
    primary key (id)) engine=InnoDB
    
使用全局的配置默认转义SQL关键字

除了使用上面的方式之外，我们还可以使用全局配置的方式开启默认关键字转义的功能，在persist.xml中加入如下的配置即可。

    <property
        name="hibernate.globally_quoted_identifiers"
        value=true"
    />
    
另外，如果我们使用spring boot的工程中集成了Hibernate，那么我们可以使用如下配置开启关键字转义的功能：

    spring.jpa.properties.hibernate.globally_quoted_identifiers=true

# 总结

如果有遇到保留的关键字作为字段名或者表名，那么我们可以利用@Column或者开启全局转义的方式避免SQL执行报错的问题。