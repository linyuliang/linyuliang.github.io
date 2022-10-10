---
title: Spring 入门(二) Spring AOP
date: 2021-01-29
excerpt_separator: "<!--more-->"
tags:  
   - Spring
   - AOP   
categories: 
   - Web开发  
author: linyuliang  
description: Spring AOP 注解的实现例子
toc: true
toc_sticky: true
---
本文是spring 入门教程（二），目标是学会Spring AOP 注解方式的使用。基本内容拷贝自以下参考资料：

参考资料：
- [Spring 参考手册（官方）](https://docs.spring.io/spring-framework/docs/current/reference/)
- [张开涛的跟我学Spring3](https://www.iteye.com/blogs/subjects/spring3)
- [面试官：什么是AOP？Spring AOP和AspectJ的区别是什么？](https://www.cnblogs.com/chaoesha/p/13037368.html)
<!-- more -->
## 理解Spring AOP
　　AOP（Aspect Orient Programming），它是面向对象编程的一种补充，主要应用于处理一些具有横切性质的系统级服务，如日志收集、事务管理、安全检查、缓存、对象池管理等。
　　AOP实现的关键就在于AOP框架自动创建的AOP代理，AOP代理则可分为静态代理和动态代理两大类，其中静态代理是指使用AOP框架提供的命令进行编译，从而在编译阶段就可生成 AOP 代理类，因此也称为编译时增强；而动态代理则在运行时借助于JDK动态代理、CGLIB等在内存中“临时”生成AOP动态代理类，因此也被称为运行时增强。
　　面向切面的编程（AOP） 是一种编程范式，旨在通过允许横切关注点的分离，提高模块化。AOP提供切面来将跨越对象关注点模块化。
　　AOP要实现的是在我们写的代码的基础上进行一定的包装，如在方法执行前、或执行后、或是在执行中出现异常后这些地方进行拦截处理或叫做增强处理。  
### AOP概念
- 连接点（Jointpoint）：表示需要在程序中插入横切关注点的扩展点，连接点可能是类初始化、方法执行、方法调用、字段调用或处理异常等等，Spring只支持方法执行连接点，在AOP中表示为“在哪里干”；
- 切入点（Pointcut）：选择一组相关连接点的模式，即可以认为连接点的集合，Spring支持perl5正则表达式和AspectJ切入点模式，Spring默认使用AspectJ语法，在AOP中表示为“在哪里干的集合”；
- 通知（Advice）：在连接点上执行的行为，通知提供了在AOP中需要在切入点所选择的连接点处进行扩展现有行为的手段；包括前置通知（before advice）、后置通知(after advice)、环绕通知（around advice），在Spring中通过代理模式实现AOP，并通过拦截器模式以环绕连接点的拦截器链织入通知；在AOP中表示为“干什么”；
- 方面/切面（Aspect）：横切关注点的模块化，比如上边提到的日志组件。可以认为是通知、引入和切入点的组合；在Spring中可以使用Schema和@AspectJ方式进行组织实现；在AOP中表示为“在哪干和干什么集合”；
- 引入（inter-type declaration）：也称为内部类型声明，为已有的类添加额外新的字段或方法，Spring允许引入新的接口（必须对应一个实现）到所有被代理对象（目标对象）, 在AOP中表示为“干什么（引入什么）”；
- 目标对象（Target Object）：需要被织入横切关注点的对象，即该对象是切入点选择的对象，需要被通知的对象，从而也可称为“被通知对象”；由于Spring AOP 通过代理模式实现，从而这个对象永远是被代理对象，在AOP中表示为“对谁干”；
- AOP代理（AOP Proxy）：AOP框架使用代理模式创建的对象，从而实现在连接点处插入通知（即应用切面），就是通过代理来对目标对象应用切面。在Spring中，AOP代理可以用JDK动态代理或CGLIB代理实现，而通过拦截器模型应用切面。
- 织入（Weaving）：织入是一个过程，是将切面应用到目标对象从而创建出AOP代理对象的过程，织入可以在编译期、类装载期、运行期进行。
### AspectJ是什么
AspectJ是一个易用的功能强大的AOP框架
AspectJ全称是Eclipse AspectJ， 其官网地址是：http://www.eclipse.org/aspectj/ 

引用官网描述：
AspectJ是什么:
- a seamless aspect-oriented extension to the Javatm programming language（一种基于Java平台的面向切面编程的语言）
- Java platform compatible（兼容Java平台，可以无缝扩展）
- easy to learn and use（易学易用）
AspectJ能做什么:
　　clean modularization of crosscutting concerns, such as error checking and handling, synchronization, context-sensitive behavior, performance optimizations, monitoring and logging, debugging support, and multi-object protocols。
　　干净的模块化横切关注点（也就是说单纯，基本上无侵入），如错误检查和处理，同步，上下文敏感的行为，性能优化，监控和记录，调试支持，多目标的协议。

　　可以单独使用，也可以整合到其它框架中。
　　单独使用AspectJ时需要使用专门的编译器ajc。java的编译器是javac，AspectJ的编译器是ajc，aj是首字母缩写，c即compiler。
### AspectJ和Spring AOP的区别
#### Spring AOP
Spring AOP 属于动态织入，通过动态代理实现
1. 基于动态代理来实现，默认如果使用接口的，用JDK提供的动态代理实现，如果是方法则使用CGLIB实现
2. Spring AOP需要依赖IOC容器来管理，并且只能作用于Spring容器，使用纯Java代码实现
3. 在性能上，由于Spring AOP是基于动态代理来实现的，在容器启动时需要生成代理实例，在方法调用上也会增加栈的深度，使得Spring AOP的性能不如AspectJ的那么好
#### AspectJ
- AspectJ属于静态织入，通过修改代码来实现，有如下几个织入的时机：
  1. 编译期织入（Compile-time weaving）： 如类 A 使用 AspectJ 添加了一个属性，类 B 引用了它，这个场景就需要编译期的时候就进行织入，否则没法编译类 B。
  2. 编译后织入（Post-compile weaving）： 也就是已经生成了 .class 文件，或已经打成 jar 包了，这种情况我们需要增强处理的话，就要用到编译后织入。
  3. 类加载后织入（Load-time weaving）： 指的是在加载类的时候进行织入，要实现这个时期的织入，有几种常见的方法。
     1. 自定义类加载器来干这个，这个应该是最容易想到的办法，在被织入类加载到 JVM 前去对它进行加载，这样就可以在加载的时候定义行为了。
     2. 在 JVM 启动的时候指定 AspectJ 提供的 agent：-javaagent:xxx/xxx/aspectjweaver.jar。
- AspectJ可以做Spring AOP干不了的事情，它是AOP编程的完全解决方案，Spring AOP则致力于解决企业级开发中最普遍的AOP（方法织入）。而不是成为像AspectJ一样的AOP方案。
- 因为AspectJ在实际运行之前就完成了织入，所以说它生成的类是没有额外运行时开销的。
#### 对比总结
  ![Spring AOP vs AspectJ](/images/20210129/Spring%20AOP%20vs%20AspectJ.jpg)
## Spring AOP 注解实现实例
　　下面是一个AOP拦截有注解的方法，进行横向增强的例子。例子直接是网上github找的慕课的代码：
### 1. 引入AspectJ面向切面依赖
在POM.xml中引入aspectJ需要的jar包
```xml
<dependency>
   <groupId>org.aspectj</groupId>
   <artifactId>aspectjweaver</artifactId>
   <version>1.9.6</version>
   <scope>runtime</scope>
</dependency>
<dependency>
   <groupId>org.aspectj</groupId>
   <artifactId>aspectjrt</artifactId>
   <version>1.9.6</version>
</dependency>
```
### 2. @AspectJ 启用切面支持
新建Spring配置类,`@EnableAspectJAutoProxy`注解启用切面支持
```java
package org.example.spring.aop;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.EnableAspectJAutoProxy;

@Configuration
@EnableAspectJAutoProxy
public class AopConfiguration {
}

```
### 3. 创建一个将要被拦截的方法注解
```java
package org.example.spring.aop.aspectj;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface MoocMethod {

    String value();

}
```
### 4. 创建一个将要拦截的业务类
```java
package org.example.spring.aop.aspectj.biz;

import org.example.spring.aop.aspectj.MoocMethod;
import org.springframework.stereotype.Service;

@Service
public class MoocBiz {

    @MoocMethod("MoocBiz save with MoocMethod.")
    public String save(String arg) {
        System.out.println("MoocBiz save : " + arg);
//		throw new RuntimeException(" Save failed!");
        return " Save success!";
    }

}
```
### 5. 申明一个切面（Aspect）
要注意的是，在Spring AOP中，切面aspect不能成为其他切面的通知的目标，有`@Aspect`注解的类，会被标记为一个切面，并排除被自动代理。
具体的`@Pointcut`可用的拦截表达式，可参考[【第六章】 AOP 之 6.5 AspectJ切入点语法详解 ——跟我学spring3](https://www.iteye.com/blog/jinnianshilongnian-1420691),要注意的是，Spring AOP仅支持拦截方法。  
```java
package org.example.spring.aop.aspectj;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

@Component
@Aspect
public class MoocAspect {

    //申明一个切入点pointcut()，拦截org.example.spring.aop.aspectj.biz这个包下面的以Biz结尾的类的所有方法
    @Pointcut("execution(* org.example.spring.aop.aspectj.biz.*Biz.*(..))")
    public void pointcut() {}

    //申明第二个切入点bizPointcut()，拦截org.example.spring.aop.aspectj.biz这个包下面的所有方法
    @Pointcut("within(org.example.spring.aop.aspectj.biz.*)")
    public void bizPointcut() {}


//    @Pointcut("@annotation(org.example.spring.aop.aspectj.MoocMethod)")
//    public void annotationPointcut() {}

    //在pointcut()拦截到后，执行方法前置通知
    @Before("pointcut()")
    public void before() {
        System.out.println("Before.");
    }
    //拦截pointcut()里面的方法，且该方法仅有一个String类型的arg名称的参数，执行方法前置通知
    @Before("pointcut() && args(arg)")
    public void beforeWithParam(String arg) {
        System.out.println("BeforeWithParam." + arg);
    }

    //拦截pointcut()里面的方法，且包含指定注解MoocMethod的方法，执行方法前置通知
    @Before("pointcut() && @annotation(moocMethod)")
    public void beforeWithAnnotaion(MoocMethod moocMethod) {
        System.out.println("BeforeWithAnnotation." + moocMethod.value());
    }

//    @Before("annotationPointcut()")
//    public void beforeWithMoocMethodAnnotation() {
//        System.out.println("beforeWithMoocMethodAnnotation.");
//    }

    //拦截bizPointcut()里面的方法，执行方法后置返回通知
    @AfterReturning(pointcut="bizPointcut()", returning="returnValue")
    public void afterReturning(Object returnValue) {
        System.out.println("AfterReturning : " + returnValue);
    }

    //拦截pointcut()里面的方法，执行方法后置异常通知
    @AfterThrowing(pointcut="pointcut()", throwing="e")
    public void afterThrowing(RuntimeException e) {
        System.out.println("AfterThrowing : " + e.getMessage());
    }

    //拦截pointcut()里面的方法，执行方法后置最终通知，类似finally
    @After("pointcut()")
    public void after() {
        System.out.println("After.");
    }

    //拦截pointcut()里面的方法，执行方法环绕通知
    @Around("pointcut()")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("Around 1.");
        Object obj = pjp.proceed();
        System.out.println("Around 2.");
        System.out.println("Around : " + obj);
        return obj;
    }

}
```
### 6. 创建应用，并执行，查看效果
```java
package org.example.spring.aop;

import org.example.spring.aop.aspectj.biz.MoocBiz;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class AopApplicationAnnotated {
    public static void main (String... args) {
        ApplicationContext ctx = new AnnotationConfigApplicationContext("org.example.spring.aop");
        MoocBiz moocBiz = ctx.getBean("moocBiz", MoocBiz.class);
        moocBiz.save("123456");
    }
}
```
