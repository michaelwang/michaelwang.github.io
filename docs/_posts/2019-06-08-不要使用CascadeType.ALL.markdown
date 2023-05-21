---
layout: post
title:  "不要使用CascadeType.ALL"
date:   2020-01-23 11:01:11 +0900
categories: hibernate
---

## 概述

在使用Hibernate级连设置的时候，我们可以设置CascadeType.ALL, 这样如果删除了父对象，就会导致相关的子对象也会被删除。而如果子对象是可以独立存在，那么就不能被删除，本文主要介绍设置了CascadeType.ALL之后，父子对象被级连删除的详细情况和应该如何设置级连属性。

## 详细介绍

假设我们有一个Student实体和一个Course实体，一个学生会选修多个课程，一个课程可能会被多个学生选择，所以我们这里就可以有如下的实体定义：

    @Entity
    @Setter
    @Getter
    @EqualsAndHashCode
    public class Student {

        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private long id;

        private String name;

        @ManyToOne
        @JoinColumn(name = "school_id")
        private School school;

        @ManyToMany(cascade = {
                CascadeType.ALL
        })
        @JoinTable(
            name = "student_course",
            joinColumns = @JoinColumn(name = "student_id"),
            inverseJoinColumns = @JoinColumn(name = "course_id")
        )
        private Set<Course> courses = new Hash<Course>();

        public void addCourse(Course course) {
            courses.add(course);
            course.getStudents().add(this);
        }

        public void deleteCourse(Course course) {
            courses.remove(course);
            course.getStudents().remove(this);
        }
    }


    @Entity
    @Getter(AccessLevel.PUBLIC)
    @Setter(AccessLevel.PUBLIC)
    public class Course {

            @Id
            @GeneratedValue(strategy = GenerationType.IDENTITY)
            private long id;

            private String name;

            @ManyToMany(mappedBy = "courses")
            private Set<Student> students = new HashSet<Student>();

       }

 Student实体中的course字段被设置为CascadeType.ALL，也就是Student实体的任何变动只要和Courses的相关，都会影响到Courses实体并被保存到DB中，我们通过如下的代码进行验证：

        tx = session.beginTransaction();
        Query query = session.createQuery("FROM Student where name = :studentName");
        query.setParameter("studentName", "name_1");
        student = (Student) query.getResultList().get(0);
        session.remove(student);
        tx.commit();

运行如上的代码产生如下的日志：

    Hibernate: 
        select
            student0_.id as id1_2_,
            student0_.name as name2_2_,
            student0_.school_id as school_i3_2_ 
        from
            Student student0_ 
        where
            student0_.name=?
    Hibernate: 
        delete 
        from
            student_course 
        where
            student_id=?
    Hibernate: 
        delete 
        from
            Course 
        where
            id=?
    Hibernate: 
        delete 
        from
            Course 
        where
            id=?
    Hibernate: 
        delete 
        from
            Student 
        where
            id=?

从上面的日志看出Student实体在被删除之前，Hibernate会删除和它相关的student_course、course这些表的记录，然后再删除student。这个做法在目前的场景来看是不合理的，因为课程可以独立存在，不一定非要依赖Student存在而存在，因此设置CascadeType.ALL是不合理的。

我么将Student的course属性调整如下的情况：

     @ManyToMany(cascade = {
                CascadeType.PERSIST,
                CascadeType.MERGE
        })
        @JoinTable(
            name = "student_course",
            joinColumns = @JoinColumn(name = "student_id"),
            inverseJoinColumns = @JoinColumn(name = "course_id")
        )
        private Set<Course> courses = new HashSet<Course>();

在如上的设置中，我们调整了只有在merge和persist的时候，才进行级连的更新，其他情况都不会级连更新。于是我们再运行上面的测试代码，产生的日志如下：

    Hibernate:  
        select
            student0_.id as id1_2_,
            student0_.name as name2_2_,
            student0_.school_id as school_i3_2_ 
        from
            Student student0_ 
        where
            student0_.name=?
    Hibernate: 
        delete 
        from
           student_course 
        where
            student_id=?
    Hibernate: 
        delete 
        from
            Student 
        where
            id=?

可以看出只删除了Student实体和Course实体的映射关系和Student实体，并没有删除Course实体。所以这样的设置符合我们的场景要求。

## 结论

我们在使用CascadeType.ALL时需要小心，因为它会默认删除父对象相关的所有子对象，而这样的做法可能会导致误删
我们使用CascadeType.Persis和CascadeType.MERGE可以在父对象保存，更新的时候将子对象级连更新到数据库，这两个配置可以作为大部分场景中的设置。
