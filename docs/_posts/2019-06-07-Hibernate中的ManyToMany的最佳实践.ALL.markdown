---
layout: post
title:  "Hibernate中的ManyToMany的最佳实践"
date:   2019-06-07 11:01:11 +0900
categories: 方法论
---

## 概述

本文将介绍如何高效的使用Hibernate的ManyToMany。
## 具体说明

假设我们有Student实体和Course实体，一个学生可以修多个课程，一个课程会有多个学生来学习。因此我们有如下的实体定义：

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
                CascadeType.PERSIST,
                CascadeType.MERGE
        })
        @JoinTable(
            name = "student_course",
            joinColumns = @JoinColumn(name = "student_id"),
            inverseJoinColumns = @JoinColumn(name = "course_id")
        )
        private List<Course> courses = new ArrayList<Course>();

        public void addCourse(Course course) {
            courses.add(course);
            course.getStudents().add(this);
        }

        public void deleteCourse(Course course) {
            courses.remove(course);
            course.getStudents().remove(this);
        }
    }

如上的代码有如下的要点：
1. 因为一个学生可以修多个课程，因此在上面的Student实体定义中，我们使用了List来定义courses属性；
2. Student和Course是双向的绑定关系，Student是ManyToMany关系的持有方，所以Student提供addCourse和deleteCourse才能保持Student和Course的关系同步；
3.级连操作设置不要设置CascadeType.ALL，因为这样会默认继承Cascade.Remove，这样会导致如果Student实体被删除，会同时删除相关联Course实体，而Course其实是可以不依赖Student实体独立存在的，所以不能设置CascadeType.ALL[参考我之前的文章]({post_url 2019-06-08-不要使用CascadeType.ALL})；

 
Course实体的定义如下：

    @Entity
    @Getter(AccessLevel.PUBLIC)
    @Setter(AccessLevel.PUBLIC)
    public class Course {

        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private long id;

        private String name;

        @ManyToMany(mappedBy = "courses",
                cascade = {CascadeType.MERGE,CascadeType.PERSIST}
        )
        private List<Student> students = new ArrayList<Student>();

    }

通过运行如下的测试代码，将course对象从student对象中删除：

     @Test
        public void saveStudentCourse() {
            Student student = new Student();
            student.setName("name_1");

            Course course = new Course();
            course.setName("c1");
            student.addCourse(course);

            course = new Course();
            course.setName("c2");
            student.addCourse(course);

            course = new Course();
            course.setName("c3");
            student.addCourse(course);

            Transaction tx = session.beginTransaction();
            session.persist(student);
            tx.commit();

            tx = session.beginTransaction();
            student.getCourses().remove(course);
            tx.commit();

        }

会产生如下的SQL:

    Hibernate: 
        insert 
        into
            Student
            (name, school_id) 
        values
            (?, ?)
    Hibernate: 
        insert 
        into
            Course
            (name) 
        values
            (?)
    Hibernate: 
        insert 
        into
            Course
            (name) 
        values
            (?)
    Hibernate: 
        insert 
        into
            Course
            (name) 
        values
            (?)
    Hibernate: 
        insert 
        into
            student_course
            (student_id, course_id) 
        values
            (?, ?)
    Hibernate: 
        insert 
        into
            student_course
            (student_id, course_id) 
        values
            (?, ?)
    Hibernate: 
        insert 
        into
            student_course
            (student_id, course_id) 
        values
            (?, ?)
    Hibernate: 
        delete 
        from
            student_course 
        where
            student_id=?
    Hibernate: 
        insert 
        into
            student_course
            (student_id, course_id) 
        values
            (?, ?)
    Hibernate: 
        insert 
        into
            student_course
            (student_id, course_id) 
        values
            (?, ?)

通过日志我们可以看出如下两点：
1. 我们保存student实例的同时会级连保存新创建出来的course实体，这是因为我们在Student实体的级连中设置了:Cascade.Persist
2. 在删除student的某一个course的时后，Hibernate会把所有student和course的中间表中的记录都删除掉之后，然后重新插入还保留的数据，这种方式并不高效，因为它生成了额外的代码。

 如果我们把Student中的courses属性的类型调整为Set之后，看看会生成怎样的SQL, 具体调整Student如下：

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
            CascadeType.PERSIST,
            CascadeType.MERGE
        })
        @JoinTable(
            name = "student_course",
            joinColumns = @JoinColumn(name = "student_id"),
            inverseJoinColumns = @JoinColumn(name = "course_id")
        )
        private Set<Course> courses = new HashSet<Course>();

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

        @ManyToMany(mappedBy = "courses",
            cascade = {CascadeType.MERGE,CascadeType.PERSIST}
        )
        private Set<Student> students = new HashSet<Student>();

    }

再运行上面的测试代码，删除Student中的某一个course对象，于是生成了如下的SQL：

    Hibernate: 
        insert 
        into
            Student
            (name, school_id) 
        values
            (?, ?)
    Hibernate: 
        insert 
        into
            Course
            (name) 
        values
            (?)
    Hibernate: 
        insert 
        into
            Course
            (name) 
        values
        (?)
    Hibernate: 
        insert 
        into
            Course
            (name) 
        values
            (?)
    Hibernate: 
        insert 
        into
            student_course
            (student_id, course_id) 
        values
            (?, ?)
    Hibernate: 
        insert 
        into
            student_course
            (student_id, course_id) 
        values
            (?, ?)
    Hibernate: 
        insert 
        into
            student_course
            (student_id, course_id) 
         values
            (?, ?)
    Hibernate: 
        delete 
        from
            student_course 
        where
            student_id=? 
            and course_id=?

可以看出删除的course只生成了一个sql语句，因此符合我们的预期。

## 总结
1. 为了高效的使用ManyToMany标注，我们最好在表示集合属性对象的时候，使用Set而不是 List，否则 Hibernate会多生成冗余的语句；
2. 关于级连设置，我们最好不要设置为CascadeType.ALL，因为这样会默认继承CascadeType.Remove，这样会删除被关联的对象
