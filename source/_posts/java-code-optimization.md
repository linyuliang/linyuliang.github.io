---
title: Java代码性能优化案例
date: 2017-11-03
excerpt_separator: "<!--more-->"
tags:
  - java
  - 性能优化
categories: 
  - Java
keywords: [java,性能优化]
description: 实际代码梳理过程中遇到的一些代码优化案例记录。
toc: true
toc_sticky: true
---

记录工作中，工程代码整理过程中遇到的一些性能需要优化的写法案例。部分案例，阿里的《阿里巴巴Java开发规约》扫描插件 [p3c](https://github.com/alibaba/p3c) 会自动检查提示，并提供示范代码，本文遇到的阿里提示优化，都直接引用阿里的插件提示代码。强烈建议大家在编译，格式化，提交和打包的时候，使用代码检查插件和配置，最好直接集成到开发工具和编译脚本上。个人集成的代码检查插件有findbugs,pmd,checkstyle,p3c（还有代码分析类插件此处略）。

<!-- more -->
## 字符串处理
1. 字符串比较用equals，和字符串常量比较，字符串常量应该放在前面。原因有：

    可以防止字符串为空的空指针异常
    可以替换
    ```java
     str!=null && str.equals("abc")
     ```
     的两个算子，提升效率

错误写法：
```java
splitLine.equals("online")
```
正确写法
```java
"online".equals(splitLine)
```

## 日志输出优化
1. 日志logger对象应尽量定义为常量，或者单例对象的一个属性，不要定义成一个方法变量
错误写法：
```java
public class XxxClass {
  public static void method(String input) {
    Logger LOGGER = LoggerFactory.getLogger(ActionParserHelper.class);
    ...
  }
}
```
正确写法
```java
public class XxxClass {
  private static Logger LOGGER = LoggerFactory.getLogger(XxxClass.class);
  public static void method(String input) {
    ...
  }
}
```
2. 使用占位符{}来格式化输出日志，而不是用拼接字符串的方式。原因有：

      可读性：使用占位符的方式，代码更清晰明了

      效率：使用占位符的方式，在日志不需要输出时（例如：下面日志的日志级别配置的info，debug级别日志不会输出），不会触发字符串的拼接操作。

  错误写法：
  ```java
  log.debug(showName + "\r\n" + showCli);
  ```
  正确写法
  ```java
  log.debug("cliLog: {} \r\n {}", showName, showCli);
  ```
3. 异常的日志打印，应该调用log的异常打印方法,最后一个参数为异常的实例，不能直接打印toString，或者拼接字符串

  错误写法：
  ```java
  log.error(" " + e);
  ```
  正确写法
  ```java
  log.error("syslog parse error：", e);
  ```

## 循环优化
1. 在循环内部不要新建固定的对象，将固定的对象的新建放到循环体外面。原因有：

    效率：重复创建对象没有必要，消耗性能。

错误写法：
```java
String pantternStr = "rate (.*?) ";

for (nline = 0; nline < strrateLine.length; nline++) {
 Pattern p = Pattern.compile(pantternStr);
 Matcher m = p.matcher(strrateLine[nline]);
 ...
}
```
正确写法
```java
String pantternStr = "rate (.*?) ";
Pattern p = Pattern.compile(pantternStr);
for (nline = 0; nline < strrateLine.length; nline++) {
 Matcher m = p.matcher(strrateLine[nline]);
 ...
}
```

同时，上面的正则表达式，不应该定义在方法体内，应尽量定义为常量或者字段，见正则优化

## 正则优化
1. 正则表达式的Pattern，应尽量定义为常量或者字段，不要在方法体内定义，尤其是循环体内。原因：

    定义未常量或者字段，可以利用其预编译功能，有效加快正则匹配速度。

错误写法：
```java
public class XxxClass {
  public Pattern getNumberPattern() {
      // Avoid use Pattern.compile in method body.!!!特别不要在循环体内使用.
      Pattern localPattern = Pattern.compile("[0-9]+");
      return localPattern;
  }
}
```
正确写法
```java
public class XxxClass {
    // Use precompile
    private static Pattern NUMBER_PATTERN = Pattern.compile("[0-9]+");
}
```
