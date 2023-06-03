---
layout: post
title:  "MyCAT-Spring-Boot-JPA中间件使用过程的问题和解决过程"
date:   2018-12-07 11:01:11 +0900
categories: java
---

## 概述

最近由于工作的关系，需要在使用Spring Boot + JPA的工程中使用用MyCAT，但是使用过程中，发生了如下的问题

    java.sql.SQLException: Connection is read-only. Queries leading to data modification are not allowed

首先说明一点，在使用MyCAT中间件之前不存在该问题。在 Spring的事务管理中，会为当前的请求创建一个EntityManager对象，以及Transaction对象， 并将它们保存在当前线程可见的ThreadLocal对象中，当一个Http请求触发需要先后执行 两个独立的数据库事务，第二个事务会从ThreadLocal中拿取第一个事务创建的EntityManager对象， 而EntityManager对象同时包含了数据库的Connection对象，也就是说Connection对象被复用了， 如果第一个事务为读数据请求，Spring的事务管理器会将Connection对象的状态设置为read only， 当第一个事务结束之后，第二个事务启动，并且需要写数据，Spring 就需要将当前的链接状态修改为read write状态，当Spring发送修改链接属性的指令的时候， 由于MyCAT中间件没有传递该指令到MySQL服务端，而导致该问题发生。

下面是问题的详细定位和分析的过程。
分析过程

因为在使用MyCAT之前，没有发生该问题，而根据错误提示来看，是写数据库请求使用了只读状态的数据库链接导致的问题。 结合问题发生时后的代码：(注意，如下不是实际的代码，但是是按照发生问题的代码逻辑写的)

     public void saveUserRole(User user, String roleId) {
        //omit some code
        Role role = findBy(Long.parseLong(roleId));
        //omit some code
        roleRepository.updateUserRole(user.getId(), Long.parseLong(roleId));
      }
    
      public Role findBy(long id) {
        return roleRepository.findOne(id);
      }
    
因为saveUserRole方法并没有使用@Transaction标注， 所以roleRepository.findOne方法会被Spring设置为以read only状态的Connection执行事务。 Spring是如何做到这一点的？分析如下： 所有应用定义的Repository都被Spring委托给SimpleJpaRepository来执行的，而SimpleJpaRepository默认 给所有的读数据的方法设置read only状态的链接去执行事务，所有写数据的方法都被设置read write状态的链接去执行查询， SimpleJpaRepository的代码如下：

    @Repository
    @Transactional(readOnly = true)
    public class SimpleJpaRepository<T, ID extends Serializable>
                    implements JpaRepository<T, ID>, JpaSpecificationExecutor<T> {
    
    
            // omit somme code
    
            /*
             * (non-Javadoc)
             * @see org.springframework.data.repository.query.QueryByExampleExecutor#findOne(org.springframework.data.domain.Example)
             */
            @Override
            public <S extends T> S findOne(Example<S> example) {
                    try {
                            return getQuery(new ExampleSpecification<S>(example), example.getProbeType(), (Sort) null).getSingleResult();
                    } catch (NoResultException e) {
                            return null;
                    }
            }
    
    
            /*
             * (non-Javadoc)
             * @see org.springframework.data.repository.CrudRepository#delete(java.io.Serializable)
             */
            @Transactional
            public void delete(ID id) {
    
                    Assert.notNull(id, ID_MUST_NOT_BE_NULL);
    
                    T entity = findOne(id);
    
                    if (entity == null) {
                            throw new EmptyResultDataAccessException(
                                            String.format("No %s entity with id %s exists!", entityInformation.getJavaType(), id), 1);
                    }
    
                    delete(entity);
            }
    
    }
    
所有的写数据的方法都有@Transaction标注，而该标注默认给数据库链接设置read write状态，具体代码如下：

        /**
         * {@code true} if the transaction is read-only.
         * <p>Defaults to {@code false}.
         * <p>This just serves as a hint for the actual transaction subsystem;
         * it will <i>not necessarily</i> cause failure of write access attempts.
         * A transaction manager which cannot interpret the read-only hint will
         * <i>not</i> throw an exception when asked for a read-only transaction
         * but rather silently ignore the hint.
         * @see org.springframework.transaction.interceptor.TransactionAttribute#isReadOnly()
         */
        boolean readOnly() default false;

所以猜测当前的问题是由于先调用了roleRepository.findOne方法，所以当前的链接被设置为read only状态， 接着调用roleRepository.updateUserRole的方法时， Spring需要将链接对象状态从read only调整为read write时，实际上该调整没有执行成功。

为了验证上述Spring管理事务时数据库链接状态切换的猜测，先不通过MyCAT启动程序进行求证， 项目使用的是log4j2,因此在log4j2.xml做添加如下的配置，目的是开启Spring Transaction和JPA的相关的日志输出，

    <logger name="org.springframework.orm.jpa" level="debug" additivity="false">
               <appender-ref ref="FILE"/>
               <appender-ref ref="STDOUT"/>
           </logger>
    
           <logger name="org.hibernate.SQL" level="debug" additivity="false">
               <appender-ref ref="FILE"/>
               <appender-ref ref="STDOUT"/>
           </logger>
    
           <logger name="org.hibernate.type.descriptor.sql.BasicBinder" level="trace" additivity="false">
               <appender-ref ref="FILE"/>
               <appender-ref ref="STDOUT"/>
     </logger>
    
然后再对应用进行一次http请求，并有如下的日志输出：

    2018-10-19 21:03:29.198 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Not closing pre-bound JPA EntityManager after transaction
    2018-10-19 21:03:29.229 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Found thread-bound EntityManager [org.hibernate.jpa.internal.EntityManagerImpl@435bf90c] for JPA transaction
    2018-10-19 21:03:29.231 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Creating new transaction with name [org.springframework.data.jpa.repository.support.QueryDslJpaRepository.findOne]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT,readOnly; ''
    2018-10-19 21:03:29.232 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Exposing JPA transaction as JDBC transaction [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@59109eec]
    2018-10-19 21:03:29.235 DEBUG localhost.localdomain --- [tp1596891431-42] o.h.SQL                                  :
        select
            role0_.id as id1_11_0_,
            role0_.area_id as area_id2_11_0_,
            role0_.is_reserved as is_reser3_11_0_,
            role0_.name as name4_11_0_,
            role0_.status as status5_11_0_,
            privileges1_.roleid as roleid1_2_1_,
            privilege2_.mod_priv_id as mod_priv2_2_1_,
            privilege2_.mod_priv_id as mod_priv1_1_2_,
            privilege2_.priv_id as priv_id2_1_2_,
            privilege2_.module_id as module_i3_1_2_,
            metaprivil3_.meta_priv_id as meta_pri1_0_3_,
            metaprivil3_.meta_priv_code as meta_pri2_0_3_,
            metaprivil3_.methods as methods3_0_3_,
            metaprivil3_.meta_priv_name as meta_pri4_0_3_,
            module4_.id as id1_10_4_,
            module4_.url as url2_10_4_,
            module4_.name as name3_10_4_,
            module4_.parent_id as parent_i4_10_4_
        from
            privilege_roles role0_
        left outer join
            account_role_priv privileges1_
                on role0_.id=privileges1_.roleid
        left outer join
            account_module_priv privilege2_
                on privileges1_.mod_priv_id=privilege2_.mod_priv_id
        left outer join
            account_meta_privileges metaprivil3_
                on privilege2_.priv_id=metaprivil3_.meta_priv_id
        left outer join
            privilege_apps_resources module4_
                on privilege2_.module_id=module4_.id
        where
            role0_.id=?
    2018-10-19 21:03:29.238 TRACE localhost.localdomain --- [tp1596891431-42] o.h.t.d.s.BasicBinder                    : binding parameter [1] as [BIGINT] - [3]
    2018-10-19 21:03:29.242 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Initiating transaction commit
    2018-10-19 21:03:29.243 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Committing JPA transaction on EntityManager [org.hibernate.jpa.internal.EntityManagerImpl@435bf90c]
    2018-10-19 21:03:29.245 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Not closing pre-bound JPA EntityManager after transaction
    ......
    ......
    2018-10-19 21:03:29.263 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Not closing pre-bound JPA EntityManager after transaction
    2018-10-19 21:03:29.268 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Found thread-bound EntityManager [org.hibernate.jpa.internal.EntityManagerImpl@435bf90c] for JPA transaction
    2018-10-19 21:03:29.268 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Creating new transaction with name [com.shihang.usercenter.repository.RoleRepositoryImpl.updateUserRole]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT; ''
    2018-10-19 21:03:29.269 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Exposing JPA transaction as JDBC transaction [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@6a64644a]
    2018-10-19 21:03:29.312 DEBUG localhost.localdomain --- [tp1596891431-42] o.h.SQL                                  :
        UPDATE
            privilege_users_roles
        SET
            role_id = ?
        WHERE
            user_id = ?
    2018-10-19 21:03:29.316 TRACE localhost.localdomain --- [tp1596891431-42] o.h.t.d.s.BasicBinder                    : binding parameter [1] as [BIGINT] - [3]
    2018-10-19 21:03:29.317 TRACE localhost.localdomain --- [tp1596891431-42] o.h.t.d.s.BasicBinder                    : binding parameter [2] as [BIGINT] - [4]
    2018-10-19 21:03:29.321 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Initiating transaction commit
    2018-10-19 21:03:29.322 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Committing JPA transaction on EntityManager [org.hibernate.jpa.internal.EntityManagerImpl@435bf90c]
    ......
    
再进入mysql cli，进行如下设置，打开mysql的sql查询日志：

    SET GLOBAL log_output = "FILE";
    SET GLOBAL general_log_file = "/path/to/your/logfile.log";
    SET GLOBAL general_log = 'ON';
    
有如下的日志输出：

    2018-10-19T13:03:29.232251Z       485 Query    set session transaction read only
    2018-10-19T13:03:29.240716Z       485 Query    select role0_.id as id1_11_0_, role0_.area_id as area_id2_11_0_, role0_.is_reserved as is_reser3_11_0_, role0_.name as name4_11_0_, role0_.status as status5_11_0_, privileges1_.roleid as roleid1_2_1_, privilege2_.mod_priv_id as mod_priv2_2_1_, privilege2_.mod_priv_id as mod_priv1_1_2_, privilege2_.priv_id as priv_id2_1_2_, privilege2_.module_id as module_i3_1_2_, metaprivil3_.meta_priv_id as meta_pri1_0_3_, metaprivil3_.meta_priv_code as meta_pri2_0_3_, metaprivil3_.methods as methods3_0_3_, metaprivil3_.meta_priv_name as meta_pri4_0_3_, module4_.id as id1_10_4_, module4_.url as url2_10_4_, module4_.name as name3_10_4_, module4_.parent_id as parent_i4_10_4_ from privilege_roles role0_ left outer join account_role_priv privileges1_ on role0_.id=privileges1_.roleid left outer join account_module_priv privilege2_ on privileges1_.mod_priv_id=privilege2_.mod_priv_id left outer join account_meta_privileges metaprivil3_ on privilege2_.priv_id=metaprivil3_.meta_priv_id left outer join privilege_apps_resources module4_ on privilege2_.module_id=module4_.id where role0_.id=3
    2018-10-19T13:03:29.244551Z       485 Query    commit
    ...
    ...
    ...
    2018-10-19T13:03:29.262568Z       485 Query    select @@session.tx_read_only
    2018-10-19T13:03:29.262873Z       485 Query    set session transaction read write
    2018-10-19T13:03:29.318828Z       485 Query    UPDATE privilege_users_roles SET role_id = 3 WHERE user_id = 4
    2018-10-19T13:03:29.323294Z       485 Query    commit
    
我们结合代码来分析日志的输出，

其中有如下的代码片段，

      public Role findBy(long id) {
        return roleRepository.findOne(id);
      }
    
触发了如下的日志输出：

    2018-10-19 21:03:29.229 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Found thread-bound EntityManager [org.hibernate.jpa.internal.EntityManagerImpl@435bf90c] for JPA transaction
    2018-10-19 21:03:29.231 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Creating new transaction with name [org.springframework.data.jpa.repository.support.QueryDslJpaRepository.findOne]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT,readOnly; ''
    2018-10-19 21:03:29.232 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Exposing JPA transaction as JDBC transaction [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@59109eec]
    2018-10-19 21:03:29.235 DEBUG localhost.localdomain --- [tp1596891431-42] o.h.SQL                                  :
        select
            role0_.id as id1_11_0_,
            role0_.area_id as area_id2_11_0_,
            role0_.is_reserved as is_reser3_11_0_,
            role0_.name as name4_11_0_,
            role0_.status as status5_11_0_,
            privileges1_.roleid as roleid1_2_1_,
            privilege2_.mod_priv_id as mod_priv2_2_1_,
            privilege2_.mod_priv_id as mod_priv1_1_2_,
            privilege2_.priv_id as priv_id2_1_2_,
            privilege2_.module_id as module_i3_1_2_,
            metaprivil3_.meta_priv_id as meta_pri1_0_3_,
            metaprivil3_.meta_priv_code as meta_pri2_0_3_,
            metaprivil3_.methods as methods3_0_3_,
            metaprivil3_.meta_priv_name as meta_pri4_0_3_,
            module4_.id as id1_10_4_,
            module4_.url as url2_10_4_,
            module4_.name as name3_10_4_,
            module4_.parent_id as parent_i4_10_4_
        from
            privilege_roles role0_
        left outer join
            account_role_priv privileges1_
                on role0_.id=privileges1_.roleid
        left outer join
            account_module_priv privilege2_
                on privileges1_.mod_priv_id=privilege2_.mod_priv_id
        left outer join
            account_meta_privileges metaprivil3_
                on privilege2_.priv_id=metaprivil3_.meta_priv_id
        left outer join
            privilege_apps_resources module4_
                on privilege2_.module_id=module4_.id
        where
            role0_.id=?
    2018-10-19 21:03:29.238 TRACE localhost.localdomain --- [tp1596891431-42] o.h.t.d.s.BasicBinder                    : binding parameter [1] as [BIGINT] - [3]
    2018-10-19 21:03:29.242 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Initiating transaction commit
    2018-10-19 21:03:29.243 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Committing JPA transaction on EntityManager [org.hibernate.jpa.internal.EntityManagerImpl@435bf90c]
    
可以看出Entitymanager对象是从前面的线程处理逻辑中获取的，并创建了一个数据库链接状态为readOnly的事务，对应在MySQL Server，有如下的日志输出

    2018-10-19T13:03:29.232251Z       485 Query    set session transaction read only
    2018-10-19T13:03:29.240716Z       485 Query    select role0_.id as id1_11_0_, role0_.area_id as area_id2_11_0_, role0_.is_reserved as is_reser3_11_0_, role0_.name as name4_11_0_, role0_.status as status5_11_0_, privileges1_.roleid as roleid1_2_1_, privilege2_.mod_priv_id as mod_priv2_2_1_, privilege2_.mod_priv_id as mod_priv1_1_2_, privilege2_.priv_id as priv_id2_1_2_, privilege2_.module_id as module_i3_1_2_, metaprivil3_.meta_priv_id as meta_pri1_0_3_, metaprivil3_.meta_priv_code as meta_pri2_0_3_, metaprivil3_.methods as methods3_0_3_, metaprivil3_.meta_priv_name as meta_pri4_0_3_, module4_.id as id1_10_4_, module4_.url as url2_10_4_, module4_.name as name3_10_4_, module4_.parent_id as parent_i4_10_4_ from privilege_roles role0_ left outer join account_role_priv privileges1_ on role0_.id=privileges1_.roleid left outer join account_module_priv privilege2_ on privileges1_.mod_priv_id=privilege2_.mod_priv_id left outer join account_meta_privileges metaprivil3_ on privilege2_.priv_id=metaprivil3_.meta_priv_id left outer join privilege_apps_resources module4_ on privilege2_.module_id=module4_.id where role0_.id=3
    2018-10-19T13:03:29.244551Z       485 Query    commit
    
再有如下的代码片段

    roleRepository.updateUserRole(user.getId(), Long.parseLong(roleId));

触发了如下的应用日志输出：

    2018-10-19 21:03:29.263 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Not closing pre-bound JPA EntityManager after transaction
    2018-10-19 21:03:29.268 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Found thread-bound EntityManager [org.hibernate.jpa.internal.EntityManagerImpl@435bf90c] for JPA transaction
    2018-10-19 21:03:29.268 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Creating new transaction with name [com.shihang.usercenter.repository.RoleRepositoryImpl.updateUserRole]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT; ''
    2018-10-19 21:03:29.269 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Exposing JPA transaction as JDBC transaction [org.springframework.orm.jpa.vendor.HibernateJpaDialect$HibernateConnectionHandle@6a64644a]
    2018-10-19 21:03:29.312 DEBUG localhost.localdomain --- [tp1596891431-42] o.h.SQL                                  :
        UPDATE
            privilege_users_roles
        SET
            role_id = ?
        WHERE
            user_id = ?
    2018-10-19 21:03:29.316 TRACE localhost.localdomain --- [tp1596891431-42] o.h.t.d.s.BasicBinder                    : binding parameter [1] as [BIGINT] - [3]
    2018-10-19 21:03:29.317 TRACE localhost.localdomain --- [tp1596891431-42] o.h.t.d.s.BasicBinder                    : binding parameter [2] as [BIGINT] - [4]
    2018-10-19 21:03:29.321 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Initiating transaction commit
    2018-10-19 21:03:29.322 DEBUG localhost.localdomain --- [tp1596891431-42] o.s.o.j.JpaTransactionManager            : Committing JPA transaction on EntityManager [org.hibernate.jpa.internal.EntityManagerImpl@435bf90c]
    ......
    
可以看出在创建一个链接状态为非readOnly的事务，因为和上面的事务创建日志输出对比，少了readOnly，因此推测链接状态为read write， 为了验证这一点，再查看MySQL Server的日志，如下：

    2018-10-19T13:03:29.262568Z       485 Query    select @@session.tx_read_only
    2018-10-19T13:03:29.262873Z       485 Query    set session transaction read write
    2018-10-19T13:03:29.318828Z       485 Query    UPDATE privilege_users_roles SET role_id = 3 WHERE user_id = 4
    2018-10-19T13:03:29.323294Z       485 Query    commit
    
我们可以看出链接的状态被设置成了read write。

结合spring hibernate的日志和mysql query log的日志，我们可以看到 一次Http请求之后，会创建一个EntityManager对象，该对象会绑定一个数据库链接对象， 在上面的mysql log的输出中链接对象的编号都为485,该对象会被后续若干次的数据操作中复用， 第一次创建的EntityManager对象的链接状态为read only，后续的若干个事务，Spring会根据 是读请求还是写请求进行调整链接的状态。 至此上述的Spring的数据库链接状态切换过程的猜测和实际一致，因此验证就结束了，

可以看出是Spring在更改连接的状态从read only到read write时，MyCAT的中间件没有传递状态切换的指令到MySQL的服务端。

## 结论

如果不做任何调整，常规的基于Spring管理事务的工程中，不适合使用MyCAT中间件。我们需要做一些额外的调整才能将MyCAT 使用起来，至于如何调整在下一篇文章中讲解。