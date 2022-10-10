---
title: spring boot入门(一) - Hello World
date: 2020-12-15
excerpt_separator: "<!--more-->"
tags:  
   - spring boot  
categories: 
   - Web开发  
author: linyuliang  
description: spring boot快速入门，实现Hello World，简单RESTFUL API
toc: true
toc_sticky: true
---
本文是spring boot的新手入门教程（一），目标是能让人快速简单上手和了解服务端的简单Web开发。  
通过本文，可以学习使用IDEA构建一个Spring boot应用，实现经典的"Hello World!"

参考资料：
- [Spring快速入门指南](https://spring.io/quickstart)
- [Spring boot 参考手册（官方）](https://docs.spring.io/spring-boot/docs/current/reference/)
- [Spring 参考手册（官方）](https://docs.spring.io/spring-framework/docs/current/reference/)
<!-- more -->
## 前置条件
1. 安装Java开发套件JDK  
   - 使用[AdoptOpenJDK](https://adoptopenjdk.net/)：Spring推荐
   - 使用Oracle JDK：  
　　2019年4月16日当天，Oracle发布了Oracle JDK的8u211和8u212两个版本（属于JDK8系列），并从这两个版本开始将JDK的授权许可从BCL换成了OTN！也就是从这两个版本开始商用收费了！  
　　所以Oracle JDK的最后两个免费版本号为8u201和8u202，并且一般情况下，我们通常使用奇数版本，即8u201（[官方下载地址](https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html)）。  
　　奇数偶数版本差异说明见：[官方奇偶数版本说明](https://www.oracle.com/java/technologies/cpu-psu-explained.html)   
   - 阿里巴巴[Dragonwell](https://www.aliyun.com/product/dragonwell)
   - 亚马逊[corretto](https://aws.amazon.com/cn/corretto/)
   - 华为[毕昇JDK](https://www.huaweicloud.com/kunpeng/software/bishengjdk.html)

2. 安装集成开发环境
    - [IntelliJ IDEA](https://www.jetbrains.com/idea/)：选择下载IntelliJ IDEA Community Edition
    - [Eclipse](https://www.eclipse.org/downloads/packages/)：选择下载Eclipse IDE for Enterprise Java Developers  

## JDK安装
1. 下载JDK 8u201
2. 安装JDK
3. 环境变量配置
   1. 新增环境变量 `JAVA_HOME` ，值为上面安装的JDK所在目录，例如我的：`D:\Program Files\Java\jdk1.8.0_201`
   2. 新增环境变量 `CLASSPATH`，值为 `.;%JAVA_HOME%\lib;%JAVA_HOME%\lib\tools.jar` 
   3. 在系统变量中找到名为 `PATH` 的变量编辑，在后面新增 `%JAVA_HOME%\bin;` ,（注意原来Path的变量值末尾有没有 `;` 号，如果没有，先输入 `;` 号再输入上面的代码）。
4. 测试是否安装成功  
   1. 打开cmd命令框，输入 `java -version` 指令，回车运行，正确提示版本等信息，则说明环境配置正确。

## IDEA安装
1. 下载并安装IntelliJ IDEA Community Edition
2. 打开IDEA，安装插件 `spring assistant`  
　　![Spring assistant 插件安装](/images/20201215/idea-install-spring-assistant.png)

## Spring Boot快速开始  
　　本章节实现一个应用，浏览器请求本地URL：`http://localhost:8080/hello`,  
　　返回  
   ``` java
    Hello World!
   ``` 
1. 新建工程，选择 `spring assistant`，下一步  
   ![新建spring工程](/images/20201215/new-project.png)
2. 默认属性中，修改Java version, 选择 `8`，我们当前生产环境使用的jdk版本,下一步  
   ![修改java版本](/images/20201215/change-java-version.png)
3. 选择类型`WEB-->Spring Web`  
   ![选择创建WEB](/images/20201215/new-spring-web.png)
4. 在包 `com.example.demo.controller` 下创建类 `HelloController`  
   类代码如下：  
   ``` java
    package com.example.demo.controller;

    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.RequestParam;
    import org.springframework.web.bind.annotation.RestController;

    @RestController
    public class HelloController {
        @GetMapping("/hello")
        public String hello(@RequestParam(value = "name", defaultValue = "World") String name) {
            return String.format("Hello %s!", name);
        }
    }
   ```
5. 刷新maven依赖  
   ![刷新maven依赖](/images/20201215/refresh-maven.png)
6. Debug调试运行应用，注意看控制台输出日志，为tomcat容器启动，默认端口号8080  
   ![debug调试运行Spring boot](/images/20201215/debug-spring-boot.png)
7. 启动好后，本地浏览器访问URL：`http://localhost:8080/hello`  
   ![访问hello请求](/images/20201215/show-hello-world.png)

## 创建一个RESTFUL请求  
　　本章节实现一个GET请求，浏览器请求本地URL：`http://localhost:8080/greeting`,  
　　返回一个json对象
   ``` json
    {"id":1,"content":"Hello, World!"}
   ``` 
　　你可以定制`name`参数，使得返回内容根据`name`变化，例如，请求URL：`http://localhost:8080/greeting?name=User`,
　　返回一个json对象
   ``` json
    {"id":1,"content":"Hello, User!"}
   ``` 
1. 停止前面启动的应用  
   ![停止应用](/images/20201215/stop-run.png)
2. 在包`com.example.demo.model`下，新建一个Java Bean对象`Greeting`
   ![新建Greeting](/images/20201215/new-greeting-bean.png)
   类代码如下：  
   ``` java
    package com.example.demo.model;

    public class Greeting {

        private final long id;
        private final String content;

        public Greeting(long id, String content) {
            this.id = id;
            this.content = content;
        }

        public long getId() {
            return id;
        }

        public String getContent() {
            return content;
        }
    }
   ```
3. 修改前面的Controller类`HelloController`,新增响应`/greeting`请求  
   ![修改HelloController](/images/20201215/modify-hello-controller.png)
   修改后类代码如下：  
   ``` java
    package com.example.demo.controller;

    import com.example.demo.model.Greeting;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.RequestParam;
    import org.springframework.web.bind.annotation.RestController;
    import java.util.concurrent.atomic.AtomicLong;

    @RestController
    public class HelloController {

        private static final String template = "Hello, %s!";
        private final AtomicLong counter = new AtomicLong();

        @GetMapping("/hello")
        public String hello(@RequestParam(value = "name", defaultValue = "World") String name) {
            return String.format("Hello %s!", name);
        }

        @GetMapping("/greeting")
        public Greeting greeting(@RequestParam(value = "name", defaultValue = "World") String name) {
            return new Greeting(counter.incrementAndGet(), String.format(template, name));
        }
    }
   ```
4. 重启启动应用调试，如上一章节的6，7
5. 本地浏览器访问URL:`http://localhost:8080/greeting?name=Linyuliang`，每次访问`id`返回递增，`content`内容根据传入的name参数变更  
   ![浏览器请求Greeting](/images/20201215/request-greeting.png)
