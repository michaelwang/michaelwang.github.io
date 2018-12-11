## 概述

接着上一篇的文章[[http://wangzhenhua.rocks/mycat-jpa-spring-transaction.html][MyCAT-Spring-Boot-JPA中间件使用过程的问题]] 提到的问题，Spring Data 默认会对所有的读数据请求加上事务
但是在使用类似MyCAT、KingShard的读写分离中间件的时后，会将带有事务的请求都发送到主库上去，而达不到读写分离需求。
因此我们需要对Spring Data进行一定的改造，能够让读数据的请求不被事务包裹，而达到利用中间件读写分离的目的。

基于上篇文章中的分析，我们发现SimpleJapRepository是在类级别加上了@Transaction(readOnly=true)的标注，这会导致所有的
读请求都被事务包裹，所以我们需要将改造这个默认的设置。

## 如何调整的方案分析

如何改造SimpleJpaRepository，有如下几个方法
0. 通过EntityManager的方式操作数据库，绕过Spring Data读取数据库数据
1. 调整SimpleJpaRepository的源码，并重新打包Spring Data
2. 通过字节码改造SimpleJpaRepository
3. 注入自定义的SimpleJpaRepository类，并被Spring Data加载，替换SimpleJpaRepository
4. 寻找类似的问题，是否在Spring Data社区中有人提出解决方案了？

如上的方案中，方案0的尝试成本最低，所以先用该方案进行验证

## 如何解决

我们通过如下的设置，可以绕过Spring Data来查询数据

```java
  public Role findBy(long id) {
    return entityManager.find(Role.class,id);
  }
```

通过spring注入EntityManager对象，通过该对象执行数据库查询操作，
由于spring的默认传播属性是Propagation.Required，
该标注的意思是需要必须要在事务中来执行方法内的逻辑，
Propagation的定义如下：
```java
public enum Propagation {

        /**
         * Support a current transaction, create a new one if none exists.
         * Analogous to EJB transaction attribute of the same name.
         * <p>This is the default setting of a transaction annotation.
         */
        REQUIRED(TransactionDefinition.PROPAGATION_REQUIRED),

        /**
         * Support a current transaction, execute non-transactionally if none exists.
         * Analogous to EJB transaction attribute of the same name.
         * <p>Note: For transaction managers with transaction synchronization,
         * PROPAGATION_SUPPORTS is slightly different from no transaction at all,
         * as it defines a transaction scope that synchronization will apply for.
         * As a consequence, the same resources (JDBC Connection, Hibernate Session, etc)
         * will be shared for the entire specified scope. Note that this depends on
         * the actual synchronization configuration of the transaction manager.
         * @see org.springframework.transaction.support.AbstractPlatformTransactionManager#setTransactionSynchronization
         */
        SUPPORTS(TransactionDefinition.PROPAGATION_SUPPORTS),

        /**
         * Support a current transaction, throw an exception if none exists.
         * Analogous to EJB transaction attribute of the same name.
         */
        MANDATORY(TransactionDefinition.PROPAGATION_MANDATORY),

        /**
         * Create a new transaction, and suspend the current transaction if one exists.
         * Analogous to the EJB transaction attribute of the same name.
         * <p><b>NOTE:</b> Actual transaction suspension will not work out-of-the-box
         * on all transaction managers. This in particular applies to
         * {@link org.springframework.transaction.jta.JtaTransactionManager},
         * which requires the {@code javax.transaction.TransactionManager} to be
         * made available it to it (which is server-specific in standard Java EE).
         * @see org.springframework.transaction.jta.JtaTransactionManager#setTransactionManager
         */
        REQUIRES_NEW(TransactionDefinition.PROPAGATION_REQUIRES_NEW),

        /**
         * Execute non-transactionally, suspend the current transaction if one exists.
         * Analogous to EJB transaction attribute of the same name.
         * <p><b>NOTE:</b> Actual transaction suspension will not work out-of-the-box
         * on all transaction managers. This in particular applies to
         * {@link org.springframework.transaction.jta.JtaTransactionManager},
         * which requires the {@code javax.transaction.TransactionManager} to be
         * made available it to it (which is server-specific in standard Java EE).
         * @see org.springframework.transaction.jta.JtaTransactionManager#setTransactionManager
         */
        NOT_SUPPORTED(TransactionDefinition.PROPAGATION_NOT_SUPPORTED),

        /**
         * Execute non-transactionally, throw an exception if a transaction exists.
         * Analogous to EJB transaction attribute of the same name.
         */
        NEVER(TransactionDefinition.PROPAGATION_NEVER),

        /**
         * Execute within a nested transaction if a current transaction exists,
         * behave like PROPAGATION_REQUIRED else. There is no analogous feature in EJB.
         * <p>Note: Actual creation of a nested transaction will only work on specific
         * transaction managers. Out of the box, this only applies to the JDBC
         * DataSourceTransactionManager when working on a JDBC 3.0 driver.
         * Some JTA providers might support nested transactions as well.
         * @see org.springframework.jdbc.datasource.DataSourceTransactionManager
         */
        NESTED(TransactionDefinition.PROPAGATION_NESTED);
```

通过上述的定义分析，我们基于当前的场景可以选择使用NERVER，表示不需要事务包裹，我们将上面的代码调整如下的格式
```java
  @Transactional(propagation = Propagation.NEVER)
  public Role findBy(long id) {
    return entityManager.find(Role.class,id);
  }
```

理论上做了这样调整之后，查询数据的请求就不会被事务包裹，我们通过如下的方式进行验证：

1、部署应用
2、修改应用的配置和MySQL日志配置（日志配置调整和上一篇文章相同 [[http://wangzhenhua.rocks/mycat-jpa-spring-transaction.html][MyCAT-Spring-Boot-JPA中间件使用过程的问题]] )
3、模拟一次Http 请求后，
4、Spring JPA有如下的输出,可以看出没有事务相关的日志
```java
2018-10-23 09:56:56.931 DEBUG localhost.localdomain --- [tp1612360825-51] o.h.SQL                                  :
    select
        privileges0_.roleid as roleid1_2_0_,
        privileges0_.mod_priv_id as mod_priv2_2_0_,
        privilege1_.mod_priv_id as mod_priv1_1_1_,
        privilege1_.priv_id as priv_id2_1_1_,
        privilege1_.module_id as module_i3_1_1_,
        metaprivil2_.meta_priv_id as meta_pri1_0_2_,
        metaprivil2_.meta_priv_code as meta_pri2_0_2_,
        metaprivil2_.methods as methods3_0_2_,
        metaprivil2_.meta_priv_name as meta_pri4_0_2_,
        module3_.id as id1_10_3_,
        module3_.url as url2_10_3_,
        module3_.name as name3_10_3_,
        module3_.parent_id as parent_i4_10_3_,
        module4_.id as id1_10_4_,
        module4_.url as url2_10_4_,
        module4_.name as name3_10_4_,
        module4_.parent_id as parent_i4_10_4_
    from
        account_role_priv privileges0_
    inner join
        account_module_priv privilege1_
            on privileges0_.mod_priv_id=privilege1_.mod_priv_id
    left outer join
        account_meta_privileges metaprivil2_
            on privilege1_.priv_id=metaprivil2_.meta_priv_id
    left outer join
        privilege_apps_resources module3_
            on privilege1_.module_id=module3_.id
    left outer join
        privilege_apps_resources module4_
            on module3_.parent_id=module4_.id
    where
        privileges0_.roleid=?
2018-10-23 09:56:56.932 TRACE localhost.localdomain --- [tp1612360825-51] o.h.t.d.s.BasicBinder                    : binding parameter [1] as [BIGINT] - [3]
```

我们再观察MySQL Server的日志输出,也没有事务日志的输出
```java
2018-10-23T01:56:56.933240Z         7 Query	select privileges0_.roleid as roleid1_2_0_, privileges0_.mod_priv_id as mod_priv2_2_0_, privilege1_.mod_priv_id as mod_priv1_1_1_, privilege1_.priv_id as priv_id2_1_1_, privilege1_.module_id as module_i3_1_1_, metaprivil2_.meta_priv_id as meta_pri1_0_2_, metaprivil2_.meta_priv_code as meta_pri2_0_2_, metaprivil2_.methods as methods3_0_2_, metaprivil2_.meta_priv_name as meta_pri4_0_2_, module3_.id as id1_10_3_, module3_.url as url2_10_3_, module3_.name as name3_10_3_, module3_.parent_id as parent_i4_10_3_, module4_.id as id1_10_4_, module4_.url as url2_10_4_, module4_.name as name3_10_4_, module4_.parent_id as parent_i4_10_4_ from account_role_priv privileges0_ inner join account_module_priv privilege1_ on privileges0_.mod_priv_id=privilege1_.mod_priv_id left outer join account_meta_privileges metaprivil2_ on privilege1_.priv_id=metaprivil2_.meta_priv_id left outer join privilege_apps_resources module3_ on privilege1_.module_id=module3_.id left outer join privilege_apps_resources module4_ on module3_.parent_id=module4_.id where privileges0_.roleid=3
```

因此通过上面代码调整实现了查询请求不被事务包裹的目的,理论上做了这样调整之后，再经过MyCAT或者KingShard等中间件代理时，读请求就会被分配到读库上，就达到了读写分离的目的。


需要思考的问题，为什么Spring Data默认对所有的读请求加上事务？带上事务的查询请求会比不带事务的性能更好吗？
这个问题明天再进行分析。
