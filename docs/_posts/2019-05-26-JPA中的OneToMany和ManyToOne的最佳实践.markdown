---
layout: post
title:  "JPA中的OneToMany和ManyToOne的最佳实践"
date:   2019-05-26 11:01:11 +0900
categories: java
---

# 概述

我们在使用JPA的时候，会使用OneToMany表示对象之间的一对多的关系，ManyToOne表示多对一的关系。多对一和一对多只是对象关系中的正面描述和反面描述，在JPA中不一样的定义方法会有不同的SQL生成，对性能会有比较大的影响。比如我们举一个例子：一个学校会有很多的学生，很多学生都会在一个学校上学，那么我们应该会设计一个父对象School和一个子对象Student，School实体会和和Student之间有一对多的关联关系，Student实体和School实体有多对一的关联关系。在JPA中对于上述的场景描述我们会有三个不同的做法：

1. 在父对象中使用@OneToMany建立和子对象的关系关系
2. 在父对象中使用@OneToMany和@JoinColumn来建立和子对象的关系
3. 建立双向关系，在父对象中使用OneToMany，在子对象中使用ManyToOne
4. 在子对象中使用ManyToOne建立和父对象的联系

 
在父对象中使用@OneToMany建立和子对象的关系关系

单向关系是只我们只在School实体建立和Student的关系，不在Student中建立和School的关联关系，具体看如下的代码：

    import javax.persistence.*;
    import java.util.Set;
    
    @Entity
    @Setter(AccessLevel.PUBLIC)
    @Getter(AccessLevel.PUBLIC)
    @EqualsAndHashCode
    @NoArgsConstructor
    @AllArgsConstructor
    public class School {
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        @GenericGenerator(name = "native", strategy = "native")
        @Column(name = "id", updatable = false, nullable = false)
        private long id;
    
        private String name;
    
        @OneToMany(
                cascade = CascadeType.ALL,
                orphanRemoval = true
        )
        private Set<Student> students;
    
    }
    
    import lombok.EqualsAndHashCode;
    import lombok.Getter;
    import lombok.Setter;
    import org.hibernate.annotations.GenericGenerator;
    
    import javax.persistence.*;
    
    @Entity
    @Setter
    @Getter
    @EqualsAndHashCode
    public class Student {
    
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        @GenericGenerator(name = "native", strategy = "native")
        private long id;
    
        private String name;
    
    }
    
基于如上的实体定义，然后我们运行如下的测试类

    import com.wangzhenhua.jpa.entity.School;
    import com.wangzhenhua.jpa.entity.Student;
    import org.hibernate.Session;
    import org.hibernate.SessionFactory;
    import org.hibernate.Transaction;
    import org.hibernate.cfg.Configuration;
    
    import org.junit.BeforeClass;
    import org.junit.Test;
    
    import java.util.HashSet;
    import java.util.Set;
    
    
    public class PersistenceTest {
    
        private static SessionFactory factory;
    
        @BeforeClass
        public static void setup() {
    
            Configuration configuration = new Configuration();
            factory = configuration.configure().buildSessionFactory();
        }
    
        @Test
        public void saveSchool() {
            School school = new School();
            school.setName("demo");
    
            Set<Student> studentSet = new HashSet<Student>();
            Student student1 = new Student();
            student1.setName("s1");
            studentSet.add(student1);
    
            Student student2 = new Student();
            student2.setName("s2");
            studentSet.add(student2);
    
            Student student3 = new Student();
            student3.setName("s3");
            studentSet.add(student3);
            school.setStudents(studentSet);
    
            Session session = factory.openSession();
            Transaction tx = session.beginTransaction();
            session.persist(school);
            tx.commit();
            session.close();
        }
    
    }
    
会有如下的SQL生成：

    五月 26, 2019 9:54:11 下午 org.hibernate.dialect.Dialect <init>
    INFO: HHH000400: Using dialect: org.hibernate.dialect.MySQL57Dialect
    Hibernate: insert into School (name) values (?)
    Hibernate: insert into Student (name) values (?)
    Hibernate: insert into Student (name) values (?)
    Hibernate: insert into Student (name) values (?)
    Hibernate: insert into School_Student (School_id, students_id) values (?, ?)
    Hibernate: insert into School_Student (School_id, students_id) values (?, ?)
    Hibernate: insert into School_Student (School_id, students_id) values (?, ?)
    
从生成的日志中，我们可以发现有一个中间表School_Student表，这是多对多的关系中才应该生成的表，而我们目前是一对多的场景，可以看出使用单向的OneToMany标注的时候，JPA并不会生成符合我们的预期表，这里其实在 Student表中建立一个School表的外键就可以，因此最后的三个更新多对多表的SQL也没必要存在。
在父对象中使用@OneToMany和@JoinColumn来建立和子对象的关系

    import lombok.*;
    import org.hibernate.annotations.GenericGenerator;
    
    import javax.persistence.*;
    import java.util.List;
    import java.util.Set;
    
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
    
        @OneToMany(
                cascade = CascadeType.ALL,
                orphanRemoval = true
        )
        @JoinColumn(name = "school_id")
        private List<Student> students;
    
    }
    
    import lombok.EqualsAndHashCode;
    import lombok.Getter;
    import lombok.Setter;
    import org.hibernate.annotations.GenericGenerator;
    
    import javax.persistence.*;
    
    @Entity
    @Setter
    @Getter
    @EqualsAndHashCode
    public class Student {
    
        @Id
        @GeneratedValue(strategy = GenerationType.SEQUENCE)
        private long id;
    
        private String name;
    }
    
如上的实体定义会生成如下的sql语句：

    Hibernate: insert into School (name, id) values (?, ?)
    Hibernate: insert into Student (name, id) values (?, ?)
    Hibernate: insert into Student (name, id) values (?, ?)
    Hibernate: insert into Student (name, id) values (?, ?)
    Hibernate: update Student set school_id=? where id=?
    Hibernate: update Student set school_id=? where id=?
    Hibernate: update Student set school_id=? where id=?
    
可以看出这里没有生成多对多的中间表，而是在student表中多了一个school_id的字段，要比上面生成的sql要精简了很多，但是问题是这里创建student的school_id的值是通过update语句更新进去的。因此这还不是一个最佳的sql语句。
建立双向关系，在父对象中使用OneToMany，在子对象中使用ManyToOne

所谓双向关系，就是在School实体中利用OneToMany关系建立和Student实体的关系，然后再在Student实体中利用ManyToOne标注建立和School的实体关系，具体看如下的代码：

    import lombok.*;
    import org.hibernate.annotations.GenericGenerator;
    
    import javax.persistence.*;
    import java.util.Set;
    
    @Entity
    @Setter(AccessLevel.PUBLIC)
    @Getter(AccessLevel.PUBLIC)
    @EqualsAndHashCode
    @NoArgsConstructor
    @AllArgsConstructor
    public class School {
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        @GenericGenerator(name = "native", strategy = "native")
        @Column(name = "id", updatable = false, nullable = false)
        private long id;
    
        private String name;
    
        @OneToMany(
                mappedBy = "school",
                cascade = CascadeType.ALL,
                orphanRemoval = true
        )
        private Set<Student> students;
    
    }
    
    
    
    import lombok.EqualsAndHashCode;
    import lombok.Getter;
    import lombok.Setter;
    import org.hibernate.annotations.GenericGenerator;
    
    import javax.persistence.*;
    
    @Entity
    @Setter
    @Getter
    @EqualsAndHashCode
    public class Student {
    
        @Id
        @GeneratedValue(strategy = GenerationType.SEQUENCE)
        private long id;
    
        private String name;
    
        @ManyToOne
        @JoinColumn(name = "school_id")
        private School school;
    
    }
    
运行上面的单元测试后，生成如下的SQL

    Hibernate: insert into School (name) values (?)
    Hibernate: insert into Student (name, school_id) values (?, ?)
    Hibernate: insert into Student (name, school_id) values (?, ?)
    Hibernate: insert into Student (name, school_id) values (?, ?)
    
我们可以看出这个结果是我们预期生成的，并没有额外的SQL产生。但是这里还存在的问题是在School中定义的OneToMany有导致内存泄漏的风险，因为我们没有办法控制它获取数据的数量，即使我们使用了懒加载，也有内存泄漏的问题。
 
在子对象中使用ManyToOne建立和父对象的联系

因为我们使用OneToMany有内存溢出的风险，因为JPA没有提供可以控制获取记录数量的参数，因此OneToMany只适合在关联的子对象实体不多的场景里，在其他场景里，为了避免内存溢出的风险，我们最好只用ManyToOne来建立对象之间的关系，具体做法如下：

    import lombok.*;
    import org.hibernate.annotations.GenericGenerator;
    
    import javax.persistence.*;
    import java.util.List;
    import java.util.Set;
    
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
    
    }
    
    import lombok.EqualsAndHashCode;
    import lombok.Getter;
    import lombok.Setter;
    import org.hibernate.annotations.GenericGenerator;
    
    import javax.persistence.*;
    
    @Entity
    @Setter
    @Getter
    @EqualsAndHashCode
    public class Student {
    
        @Id
        @GeneratedValue(strategy = GenerationType.SEQUENCE)
        private long id;
    
        private String name;
    
        @ManyToOne
        @JoinColumn(name = "school_id")
        private School school;
    }
    
 

其实ManyToOne满足我们的所有一对多关系的所有新增、查询的场景，比如增加实体，可以通过如下的代码：

    @Test
        public void saveSchool() {
            School school = new School();
            school.setName("demo");
    
            List<Student> studentSet = new ArrayList<Student>();
            Student student1 = new Student();
            student1.setName("s1");
            student1.setSchool(school);
    
    
            Student student2 = new Student();
            student2.setName("s2");
            student2.setSchool(school);
    
            Student student3 = new Student();
            student3.setName("s3");
            student3.setSchool(school);
            studentSet.add(student1);
            studentSet.add(student2);
            studentSet.add(student3);
    
            Session session = factory.openSession();
            Transaction tx = session.beginTransaction();
            session.persist(school);
            session.persist(student1);
            session.persist(student2);
            session.persist(student3);
            tx.commit();
            session.close();
        }
    
会生成如下的sql 语句：

    Hibernate: insert into School (name, id) values (?, ?)
    Hibernate: insert into Student (name, school_id, id) values (?, ?, ?)
    Hibernate: insert into Student (name, school_id, id) values (?, ?, ?)
    Hibernate: insert into Student (name, school_id, id) values (?, ?, ?)
    
可以看出这里输出的sql是符合我们的预期，并没有额外的sql生成。

如果我们要查询一个School的所有学生，我们可以利用如下的HQL语句查询：

    session = factory.openSession();
    Query<Student> query = session.createQuery("select s from Student s where s.school.id = :schoolId", Student.class);
    query.setParameter("schoolId",1l);
    List<Student> resultList = query.getResultList();
    session.close();
    
会生成如下的sql语句:

    Hibernate: select student0_.id as id1_1_, student0_.name as name2_1_, student0_.school_id as school_i3_1_ from Student student0_ where student0_.school_id=?
    Hibernate: select school0_.id as id1_0_0_, school0_.name as name2_0_0_ from School school0_ where school0_.id=?
    
因此ManyToOne可以满足在一对多场景里的所有需求，因此推荐使用ManyToOne来作为一对多的关联管理。