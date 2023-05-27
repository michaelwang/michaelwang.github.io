---
layout: post
title:  "如何创建maven-plugin"
date:   2018-12-04 11:01:11 +0900
categories: java
---

本文主要介绍如何创建一个简单的maven插件

需要提前准备好如下内容

    安装好maven的环境
    安装好idea或者eclipse
    搭建好neuxs服务

首先在控制台执行命令:

    mvn archetype:generate

该命令会生成一个maven plugin的骨架项目。 在命令的执行过程中，会提示需要你选择一个项目类型，我们这里输入： maven-archetype-plugin 然后再选择生成的是什么类型的插件和对应的插件的版本号。

再利用idea导入刚生成的maven 工程，找到源码MyMojo.java，这个类是在maven工程的输出目录中创建一个 touch.txt文件。

我们先不对这个类做任何改动，让该插件可以在项目中执行，可以快速体验下如何执行一个自定义的插件的过程。

我们进入刚生成的目录 demo-touch中，然后执行：

     mvn clean install

这个命令会生成该插件jar包到本地的maven仓库中。

然后我们在当前项目中执行这个插件，预期执行完这个插件之后，我们的当前项目的输出目录 target/classes/中会有一个名为touch.txt文件。

现在进入项目的根目录开始执行命令：

    mvn com.weiyan:demo-touch:1.0-SNAPSHOT:touch

执行完后，我们发现生成目录中有一个名为touch.txt的文件存在。