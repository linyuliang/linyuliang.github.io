---
title: tomcat8的配置优化
date: 2017-12-08
excerpt_separator: "<!--more-->"
tags:
  - tomcat
categories: 
  - web容器
author: linyuliang
keywords: [tomcat,配置，优化]  
description: tomat8容器的配置优化。
toc: true
toc_sticky: true
---
使用任何一个第三方工具、库等，都需要对工具和库的配置进行了解和测试（一般公司都没有做测试，直接使用，不投入时间测试，实际上在后期，重复使用、遇到问题、以及资源浪费的开销更大），本文当然...也没有做测试。。。只是参考网上的资料进行整合，同时结合本身生产环境上的大并发web部署，讲解如何配置tomcat8的参数，使得web更安全，并且支撑更高的并发量，更大的吞吐量，同时降低系统资源使用率。

当然，tomcat中的JVM参数配置对web性能的影响最大最明显，但是并非本文的重点，JVM的调优内容也很多，应该另开一个文章来讲解。我们这里只讲解tomcat容器本身的配置，主要是server.xml的配置优化。




<!-- more -->
## 安全配置 --基本拷贝至 [Tomcat8安装， 安全配置与性能优化](http://blog.csdn.net/our_sky/article/details/51362676)
一般情况下，软件的漏洞信息和版本是相关的，因此，软件的版本号对于攻击者来说是很有价值的。所以，在使用第三方软件的时候，一定要注意隐藏软件的版本信息等。另外，有的还需要配置端口，黑白名单，权限。
1. 隐藏版本信息

  隐藏HTTP 头部的版本信息，编辑tomcat的conf目录下的server.xml文件，为Connector节点添加 server 属性

  ```xml
  <Connector port="8080" protocol="HTTP/1.1"  
                                            connectionTimeout="20000"  
                                            redirectPort="8443" server="APP Srv1.0"/>
  ```

2. 隐藏或伪装Tomcat的版本信息

  修改Tomcat 安装目录下的lib目录内的catalina.jar，修改该jar包内部的文件org/apache/catalina/util/ServerInfo.properties保存的版本信息

  ```java
  server.info=Apache Tomcat
  server.number=0.0.0.0
  server.built=Jan 1 2000 00:00:00 UTC
  ```

  提示：上面的1,2隐藏信息，包括404页面显示的版本信息等，实际都可以通过在nginx过滤掉。

3. 禁用Tomcat管理界面

  生产环境一般不适用Tomcat默认的管理界面，这类页面经常会收到全球各地发起的访问管理页面的黑客探测请求。这些页面存放在Tomcat 的webapps安装目录下，正常情况下，应当先清空tomcat安装目录下的webapps文件夹下内容。

4. 应用程序安全
  tomcat默认开启了对war热部署。为了防止被植入木马恶意攻击，我们要关闭war包自动部署。关闭自动加载最新代码（设置reloadable）

  编辑tomcat的conf目录下的server.xml文件，修改host节点的属性

  ```xml
  <Host name="localhost"  appBase="webapps"
        unpackWARs="false" autoDeploy="false" reloadable="false">
  ```

## 性能优化配置 --基本拷贝至 [Tomcat8安装， 安全配置与性能优化](http://blog.csdn.net/our_sky/article/details/51362676)

1. 连接池配置

  启用tomcat线程池，用较少的线程处理较多的访问，可以提高tomcat处理请求的能力。
  编辑tomcat的conf目录下的server.xml文件，增加Executor节点

  ```xml
  <!--
  <Executor name="tomcatThreadPool" namePrefix="catalina-exec-"
      maxThreads="150" minSpareThreads="4"/>
  -->
  <Executor
   name="tomcatThreadPool"
   namePrefix="catalina-exec-"
   maxThreads="2000"
   minSpareThreads="500"
   maxIdleTime="60000"
   prestartminSpareThreads = "true"
   maxQueueSize = "1000"
  />
  ```

  参数讲解：
  - name：线程池名称，在server.xml中必须唯一
  - namePrefix：线程池创建的线程名称前缀，后面跟着的是线程号
  - maxThreads：线程池中的最大线程数，默认是200。一般建议在 500 ~ 800，根据硬件设施和业务来判断
  - minSpareThreads：线程池中的最小线程数，默认是25
  - maxIdleTime：如果当前线程大于初始化线程，那空闲线程存活的时间，默认是60000毫秒（1分钟）
  - prestartminSpareThreads：默认是false，表示线程池启动时，是否启动最小线程数的线程，如果不等于 true，minSpareThreads 的值就没啥效果了。
  - maxQueueSize：线程池最大等待队列，超过则拒绝请求，默认是Integer.MAX_VALUE
  其他更多参数说明，参见[tomcat官方excutor说明](http://tomcat.apache.org/tomcat-8.5-doc/config/executor.html)

2. 修改默认的链接参数配置：基本拷贝至 [Tomcat 8 安装和配置、优化](https://github.com/judasn/Linux-Tutorial/blob/master/Tomcat-Install-And-Settings.md)
  - 默认值：

  ``` xml
  <Connector
      port="8080"
      protocol="HTTP/1.1"
      connectionTimeout="20000"
      redirectPort="8443"
  />
  ```

  - 修改为：

  ``` xml
  <Connector
   executor="tomcatThreadPool"
   port="8080"
   protocol="org.apache.coyote.http11.Http11Nio2Protocol"
   connectionTimeout="20000"
   maxConnections="10000"
   redirectPort="8443"
   enableLookups="false"
   acceptCount="100"
   maxPostSize="10485760"
   maxHttpHeaderSize="8192"
   compression="on"
   disableUploadTimeout="true"
   compressionMinSize="2048"
   acceptorThreadCount="2"
   compressableMimeType="text/html,text/plain,text/css,application/javascript,application/json,application/x-font-ttf,application/x-font-otf,image/svg+xml,image/jpeg,image/png,image/gif,audio/mpeg,video/mp4"
   URIEncoding="utf-8"
   processorCache="20000"
   tcpNoDelay="true"
   server="Server Version 11.0"
   />
  ```

  - 重点参数解释：
      - protocol，Tomcat 8 设置 nio2 更好：org.apache.coyote.http11.Http11Nio2Protocol（如果这个用不了，就用下面那个）
      - protocol，Tomcat 6、7 设置 nio 更好：org.apache.coyote.http11.Http11NioProtocol
      - protocol, apr模式(Apache Portable Runtime/Apache可移植运行时库)：org.apache.coyote.http11.Http11AprProtocol,Tomcat将以JNI的形式调用Apache HTTP服务器的核心动态链接库来处理文件读取或网络传输操作，从而大大地提高Tomcat对静态文件的处理性能。从操作系统级别解决异步IO问题，大幅度的提高服务器的处理和响应性能，也是Tomcat运行高并发应用的首选模式，但是部署会麻烦点，需要安装apr和native，docker也没有官方或者很流行的带apr的tomcat镜像。
      - enableLookups，禁用DNS查询
      - acceptCount，指定当所有可以使用的处理请求的线程数都被使用时，可以放到处理队列中的请求数，超过这个数的请求将不予处理，默认设置 100
      - maxPostSize，以 FORM URL 参数方式的 POST 提交方式，限制提交最大的大小，默认是 2097152(2兆)，它使用的单位是字节。10485760 为 10M。如果要禁用限制，则可以设置为 -1。
      - acceptorThreadCount，用于接收连接的线程的数量，默认值是1。一般这个指需要改动的时候是因为该服务器是一个多核CPU，如果是多核 CPU 一般配置为 2.
      - maxHttpHeaderSize，http请求头信息的最大程度，超过此长度的部分不予处理。一般8K。
      - processorCache，协议处理器缓存处理器对象以加速性能。该参数决定了缓存多少对象。默认是200。如果不使用Servlet 3异步处理，最好等于MaxThreads的值。如果使用Servlet 3异步处理，最好使用maxthreads和预期的并发请求的最大数量（同步和异步）中的较大值。
      - maxKeepAliveRequests，默认是100，服务器保持的keep alived的连接数量，直接tomcat面向客户，使用默认值即可，如果前道使用nginx拦截，建议使用-1。另外一个相关参数是keepAliveTimeout：默认值是connectionTimeout的值（单位毫秒）。
  - 更多的配置说明见 [tomcat官方说明](http://tomcat.apache.org/tomcat-8.5-doc/config/http.html#Common_Attributes)
  - 提示：
      压缩会增加Tomcat负担，最好采用Nginx + Tomcat 或者 Apache + Tomcat 方式，压缩交由Nginx/Apache 去做。

       Tomcat 的压缩是在客户端请求服务器对应资源后，从服务器端将资源文件压缩，再输出到客户端，由客户端的浏览器负责解压缩并浏览。相对于普通的 浏览过程 HTML、CSS、Javascript和Text，它可以节省40% 左右的流量。更为重要的是，它可以对动态生成的，包括CGI、PHP、JSP、ASP、Servlet,SHTML等输出的网页也能进行压缩，压缩效率也很高。

3. 禁用 AJP（如果你服务器没有使用 Apache）

  AJP是为 Tomcat 与 HTTP 服务器之间通信而定制的协议，能提供较高的通信速度和效率。如果tomcat前端放的是apache的时候，会使用到AJP这个连接器。 默认是开启的。如果不使用apache，注释该连接器。

  ``` xml
  <!-- <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" /> -->
  ```

4. 禁用accesslog输出

  所有的访问默认都会记录在access log，导致最终硬盘溢出，没有跟踪需求没必要记录，一般也可以使用nginx的日志输出进行url访问的分析统计。（当然nginx也没有做日志分割和老化，需要额外的使用cronlog工具来做）

  ``` xml
  <!--
  <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
         prefix="localhost_access_log" suffix=".txt"
         pattern="%h %l %u %t &quot;%r&quot; %s %b" />
  -->
  ```

## JVM优化配置
  默认是要修改bin下的catalina.sh，如果使用docker部署tomcat，则传进环境变量即可。另外开文章写

## 参考资料
* [Tomcat8安装， 安全配置与性能优化](http://blog.csdn.net/our_sky/article/details/51362676)
* [Tomcat 8 安装和配置、优化](https://github.com/judasn/Linux-Tutorial/blob/master/Tomcat-Install-And-Settings.md)

## 最后
1. 嗯嗯，最后搞个微信打赏二维码玩。。。
![微信扫码捐赠](/images/linyuliang_weixin_tip.jpg)
