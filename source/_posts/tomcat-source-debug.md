---
layout: post
title: "【基于IntelliJ IDEA环境】Tomcat8源码的调试和项目部署"
date: 2017-10-20 12:32
tags: [tomcat]
category: 随笔
description: 搭建tomcat8的源码编译和调试环境，详细记录调试工程的配置过程和注意点。
---
## 前言
Tomcat作为J2EE的开源实现，其代码具有很高的参考价值，我们可以从中汲取很多的知识。比如ClassLoader机制，设计模式等。本文只描述了环境搭配方法，具体的源码分析可以参考《How Tomcat Works》 一书。Tomcat现在都到9了，虽然该书是基于5.0但还是具有一定参考意义的。

## 一，准备工作
Tomcat的构建是基于Ant和Eclipse的，然而现在很多人都喜欢IDEA+Maven的项目构建方式，所以本文将基于这个环境来搭建源码的调试。我们需要以下工具：

- Tomcat源码下载地址：https://mirrors.tuna.tsinghua.edu.cn/apache/tomcat/tomcat-8/v8.5.23/src/apache-tomcat-8.5.23-src.tar.gz

- IDEA工具：https://www.jetbrains.com/idea/download

- MAVEN：http://maven.apache.org/download.cgi

- JDK：自然不用多提了，但是要按照所选源码要求的版本，这里用的是JDK8

安装和下载这些软件包就可以开始搭建调试环境了。

## 二，项目结构
![img](/upload/images/tomcat/jiegou.png)
- 新建一个目录，比如：D:\code\tomcat8，然后将tomcat8的源码解压至该目录
- 新建catalina-home目录，然后将apache-tomcat-8.5.23-src目录下的 conf文件夹拷贝到此处，该目录结构如下
![img](/upload/images/tomcat/conf.png)
除了conf目录其他都是可选的，webapps用于我们应用默认的部署目录，work logs是启动Tomcat自动生成的，其结构跟我们下载的二进制Tomcat程序是一样的.
- 配置Maven依赖

我们采用module的形式来组织目录,首先在根目录(D:\code\tomcat8)下创建pom.xml,其内容如下:
```xml
<?xml version="1.0" encoding="UTF-8"?>    
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"    
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">    
    
    <modelVersion>4.0.0</modelVersion>    
    <groupId>gxf</groupId>    
    <artifactId>apache-tomcat-8</artifactId>    
    <name>apache-tomcat-8-source</name>    
    <version>1.0</version>    
    <packaging>pom</packaging>    
    
    <modules>    
        <module>apache-tomcat-8.5.23-src</module>    
    </modules>    
</project>    
```
这里主要指定module为Tomcat的源码目录,然后在apache-tomcat-8.5.23-src配置Tomcat源码额外的依赖，在该目录创建pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>    
<project xmlns="http://maven.apache.org/POM/4.0.0"    
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"    
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">    
    
    
    <modelVersion>4.0.0</modelVersion>    
    <groupId>org.apache.tomcat</groupId>    
    <artifactId>Tomcat8.0</artifactId>    
    <name>Tomcat8.0</name>    
    <version>8.0</version>    
    
    <build>    
        <finalName>Tomcat8.0</finalName>    
        <sourceDirectory>java</sourceDirectory>    
        <testSourceDirectory>test</testSourceDirectory>    
        <resources>    
            <resource>    
                <directory>java</directory>    
            </resource>    
        </resources>    
        <testResources>    
            <testResource>    
                <directory>test</directory>    
            </testResource>    
        </testResources>    
        <plugins>    
            <plugin>    
                <groupId>org.apache.maven.plugins</groupId>    
                <artifactId>maven-compiler-plugin</artifactId>    
                <version>2.0.2</version>    
    
                <configuration>    
                    <encoding>UTF-8</encoding>    
                    <source>1.8</source>    
                    <target>1.8</target>    
                </configuration>    
            </plugin>    
        </plugins>    
    </build>    
    
    <dependencies>  
        <dependency>  
            <groupId>org.easymock</groupId>  
            <artifactId>easymock</artifactId>  
            <version>3.5</version>  
            <scope>test</scope>  
        </dependency>  
  
        <dependency>    
            <groupId>junit</groupId>    
            <artifactId>junit</artifactId>    
            <version>4.12</version>  
            <scope>test</scope>    
        </dependency>    
        <dependency>    
            <groupId>ant</groupId>    
            <artifactId>ant</artifactId>    
            <version>1.7.0</version>    
        </dependency>    
        <dependency>    
            <groupId>wsdl4j</groupId>    
            <artifactId>wsdl4j</artifactId>    
            <version>1.6.2</version>    
        </dependency>    
        <dependency>    
            <groupId>javax.xml</groupId>    
            <artifactId>jaxrpc</artifactId>    
            <version>1.1</version>    
        </dependency>    
        <dependency>    
            <groupId>org.eclipse.jdt.core.compiler</groupId>    
            <artifactId>ecj</artifactId>    
            <version>4.6.1</version>  
        </dependency>    
    </dependencies>    
    
</project>   
```
至此基本的结构已经完成了
## 三，准备构建
### 使用IDEA打开项目
![img](/upload/images/tomcat/open.png)
注意这里是直接打开项目，定位到D:\code\tomcat8，然后选择pom.xml，idea会自动识别Maven项目。这一部完成后IDEA开始下载MAVEN依赖包，完整的结构如下：
![img](/upload/images/tomcat/whole.png)
### 配置编译环境
说明：如果编译build的时候出现Test测试代码报错，删除该代码即可。本文中的Tomcat源码util.TestCookieFilter类会报错，将其删除即可。
![img](/upload/images/tomcat/env.png)
1) 打开项目的Run/Debug配置界面，Main class设置为org.apache.catalina.startup.Bootstrap
2) 添加VM options -Dcatalina.home=catalina-home -Dcatalina.base=catalina-home   -Djava.endorsed.dirs=catalina-home/endorsed -Djava.io.tmpdir=catalina-home/temp   -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager   -Djava.util.logging.config.file=catalina-home/conf/logging.properties 
3) 选择module为Tomcat8.0（源码所在的module）
4) 点击Debug按钮启动程序
### 启动工程
我们点击调试按钮后发现会报错，如下：
![img](/upload/images/tomcat/error.png)
原来是构建tomcat-jdbc的时候报错了，这里我们手动指定版本号就可以了，修改文件D:\code\tomcat8\apache-tomcat-8.5.23-src\modules\jdbc-pool\resources\MANIFEST.MF中的Bundle-Version: @VERSION@，把Bundle-Version: 8。这里的构建可能会有点卡顿。

好了，一切顺利项目启动成功了。
```xml
"C:\Program Files\Java\jdk1.8.0_111\bin\java" -agentlib:jdwp=transport=dt_socket,address=127.0.0.1:55097,suspend=y,server=n -Dvisualvm.id=150866430461380 -Dcatalina.home=catalina-home -Dcatalina.base=catalina-home -Djava.endorsed.dirs=catalina-home/endorsed -Djava.io.tmpdir=catalina-home/temp -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djava.util.logging.config.file=catalina-home/conf/logging.properties -Dfile.encoding=UTF-8 -classpath "C:\Program Files\Java\jdk1.8.0_111\jre\lib\charsets.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\deploy.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\ext\access-bridge-64.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\ext\cldrdata.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\ext\dnsns.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\ext\jaccess.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\ext\jfxrt.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\ext\localedata.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\ext\nashorn.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\ext\sunec.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\ext\sunjce_provider.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\ext\sunmscapi.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\ext\sunpkcs11.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\ext\zipfs.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\javaws.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\jce.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\jfr.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\jfxswt.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\jsse.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\management-agent.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\plugin.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\resources.jar;C:\Program Files\Java\jdk1.8.0_111\jre\lib\rt.jar;D:\code\tomcat8\apache-tomcat-8.5.23-src\target\classes;C:\Users\gongxufan\.m2\repository\org\apache\ant\ant\1.7.0\ant-1.7.0.jar;C:\Users\gongxufan\.m2\repository\org\apache\ant\ant-launcher\1.7.0\ant-launcher-1.7.0.jar;C:\Users\gongxufan\.m2\repository\wsdl4j\wsdl4j\1.6.2\wsdl4j-1.6.2.jar;C:\Users\gongxufan\.m2\repository\javax\xml\jaxrpc-api\1.1\jaxrpc-api-1.1.jar;C:\Users\gongxufan\.m2\repository\org\eclipse\jdt\core\compiler\ecj\4.6.1\ecj-4.6.1.jar;D:\soft\IntelliJ IDEA 2017.1.4\lib\idea_rt.jar" org.apache.catalina.startup.Bootstrap  
Connected to the target VM, address: '127.0.0.1:55097', transport: 'socket'  
20-Oct-2017 13:05:22.684 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log Server version:        Apache Tomcat/@VERSION@  
20-Oct-2017 13:05:22.691 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log Server built:          @VERSION_BUILT@  
20-Oct-2017 13:05:22.691 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log Server number:         @VERSION_NUMBER@  
20-Oct-2017 13:05:22.692 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log OS Name:               Windows 10  
20-Oct-2017 13:05:22.692 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log OS Version:            10.0  
20-Oct-2017 13:05:22.692 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log Architecture:          amd64  
20-Oct-2017 13:05:22.692 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log Java Home:             C:\Program Files\Java\jdk1.8.0_111\jre  
20-Oct-2017 13:05:22.693 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log JVM Version:           1.8.0_111-b14  
20-Oct-2017 13:05:22.693 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log JVM Vendor:            Oracle Corporation  
20-Oct-2017 13:05:22.693 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log CATALINA_BASE:         D:\code\tomcat8\catalina-home  
20-Oct-2017 13:05:22.693 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log CATALINA_HOME:         D:\code\tomcat8\catalina-home  
20-Oct-2017 13:05:22.693 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -agentlib:jdwp=transport=dt_socket,address=127.0.0.1:55097,suspend=y,server=n  
20-Oct-2017 13:05:22.693 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Dvisualvm.id=150866430461380  
20-Oct-2017 13:05:22.694 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Dcatalina.home=catalina-home  
20-Oct-2017 13:05:22.694 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Dcatalina.base=catalina-home  
20-Oct-2017 13:05:22.694 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Djava.endorsed.dirs=catalina-home/endorsed  
20-Oct-2017 13:05:22.694 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Djava.io.tmpdir=catalina-home/temp  
20-Oct-2017 13:05:22.694 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager  
20-Oct-2017 13:05:22.694 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Djava.util.logging.config.file=catalina-home/conf/logging.properties  
20-Oct-2017 13:05:22.694 信息 [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Dfile.encoding=UTF-8  
20-Oct-2017 13:05:22.695 信息 [main] org.apache.catalina.core.AprLifecycleListener.lifecycleEvent The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path: [C:\Program Files\Java\jdk1.8.0_111\bin;C:\WINDOWS\Sun\Java\bin;C:\WINDOWS\system32;C:\WINDOWS;D:\soft\oracle\bin;C:\Python27\;C:\Python27\Scripts;C:\WINDOWS\system32;C:\WINDOWS;C:\WINDOWS\System32\Wbem;C:\WINDOWS\System32\WindowsPowerShell\v1.0\;C:\Program Files (x86)\ATI Technologies\ATI.ACE\Core-Static;D:\apache-maven-3.5.0\bin;C:\Program Files\Java\jdk1.8.0_111\bin;D:\soft\Git\cmd;C:\Program Files\MySQL\MySQL Server 5.5\bin;D:\soft\AndroidStudio\sdk\tools;D:\soft\AndroidStudio\sdk\platform-tools;D:\soft\nodejs\;D:\soft\AndroidStudio\sdk\tools\bin;D:\soft\SVN\bin;D:\soft\AndroidStudio\app\gradle\gradle-3.2\bin;D:\soft\gradle-4.1\bin;C:\Users\gongxufan\AppData\Local\Microsoft\WindowsApps;C:\Users\gongxufan\AppData\Roaming\npm;;.]  
20-Oct-2017 13:05:22.941 信息 [main] org.apache.coyote.AbstractProtocol.init Initializing ProtocolHandler ["http-nio-8080"]  
20-Oct-2017 13:05:23.226 信息 [main] org.apache.tomcat.util.net.NioSelectorPool.getSharedSelector Using a shared selector for servlet write/read  
20-Oct-2017 13:05:23.230 信息 [main] org.apache.coyote.AbstractProtocol.init Initializing ProtocolHandler ["ajp-nio-8009"]  
20-Oct-2017 13:05:23.232 信息 [main] org.apache.tomcat.util.net.NioSelectorPool.getSharedSelector Using a shared selector for servlet write/read  
20-Oct-2017 13:05:23.233 信息 [main] org.apache.catalina.startup.Catalina.load Initialization processed in 1361 ms  
20-Oct-2017 13:05:23.290 信息 [main] org.apache.catalina.core.StandardService.startInternal Starting service [Catalina]  
20-Oct-2017 13:05:23.290 信息 [main] org.apache.catalina.core.StandardEngine.startInternal Starting Servlet Engine: Apache Tomcat/@VERSION@  
```
## 四，项目部署
项目启动完毕我们可以测试了，在浏览器访问http://localhost:8080/就可以看到我们经典的欢迎页面了。这里前提是要把源码目录下的webapps复制到catalina-home目录。

然而当访问JSP的时候却报错了，比如访问：http://localhost:8080/index.jsp或者我们自己的项目的时候会出现JSP无法编译的错误。
```java
java.lang.NullPointerException  
    at org.apache.jasper.JspCompilationContext.getTldResourcePath(JspCompilationContext.java:536)  
    at org.apache.jasper.compiler.Parser.parseTaglibDirective(Parser.java:410)  
    at org.apache.jasper.compiler.Parser.parseDirective(Parser.java:469)  
    at org.apache.jasper.compiler.Parser.parseElements(Parser.java:1430)  
    at org.apache.jasper.compiler.Parser.parse(Parser.java:139)  
    at org.apache.jasper.compiler.ParserController.doParse(ParserController.java:227)  
    at org.apache.jasper.compiler.ParserController.parse(ParserController.java:100)  
    at org.apache.jasper.compiler.Compiler.generateJava(Compiler.java:199)  
    at org.apache.jasper.compiler.Compiler.compile(Compiler.java:356)  
    at org.apache.jasper.compiler.Compiler.compile(Compiler.java:336)  
    at org.apache.jasper.compiler.Compiler.compile(Compiler.java:323)  
    at org.apache.jasper.JspCompilationContext.compile(JspCompilationContext.java:570)  
    at org.apache.jasper.servlet.JspServletWrapper.service(JspServletWrapper.java:356)  
    at org.apache.jasper.servlet.JspServlet.serviceJspFile(JspServlet.java:396)  
    at org.apache.jasper.servlet.JspServlet.service(JspServlet.java:340)  
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:725)  
    at org.wso2.carbon.ui.JspServlet.service(JspServlet.java:155)  
    at org.wso2.carbon.ui.TilesJspServlet.service(TilesJspServlet.java:80)  
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:725)  
```
原因是我们直接启动org.apache.catalina.startup.Bootstrap的时候没有加载org.apache.jasper.servlet.JasperInitializer，从而无法编译JSP。这在Tomcat6/7是没有这个问题的。解决办法是在tomcat的源码org.apache.catalina.startup.ContextConfig中手动将JSP解析器初始化：
`context.addServletContainerInitializer(new JasperInitializer(), null);`
![img](/upload/images/tomcat/init.png)
至此，我们可以将项目直接放到Tomcat来启动调试了。本文描述的方式同样可以适用于其他版本(Tomcat6+)的Tomcat调试。

## 五，部署外部项目
默认情况下我们必须把应用程序部署到catalina-home/webapps下，如何直接部署访问外部的应用程序?我们只需要修改server.xml和web.xml配置即可。
1) 编辑catalina-home/conf/server.xml
```xml
<Host name="localhost"  appBase="webapps"  
           unpackWARs="true" autoDeploy="true">  
  
       <!-- SingleSignOn valve, share authentication between web applications  
            Documentation at: /docs/config/valve.html -->  
       <!-- 
       <Valve className="org.apache.catalina.authenticator.SingleSignOn" /> 
       -->  
  
       <!-- Access log processes all example.  
            Documentation at: /docs/config/valve.html  
            Note: The pattern used is equivalent to using pattern="common" -->  
       <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"  
              prefix="localhost_access_log" suffix=".txt"  
              pattern="%h %l %u %t "%r" %s %b" />  
       <Context path="/easyshare-web" reloadable="true" debug="0" docBase="D:\code\work\easyshare2017\easyshare-web\target\easyshare-web-1.0-SNAPSHOT" workDir="D:\code\work\easyshare2017\easyshare-web\target\easyshare-web-1.0-SNAPSHOT\work" crossContext="true" >  
       </Context>  
     </Host>  

```
在Host节点增加Context配置，path是我们访问项目时的contextPath，docBase设置为外部应用程序的绝对路径,workDir为jsp编译结果存放路径，这里是指为docBase/work
2) 编辑catalina-home/conf/web.xml（一般不需要这个步骤，如果第一步出现404再尝试开启）
```xml
<servlet>  
       <servlet-name>default</servlet-name>  
       <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>  
       <init-param>  
           <param-name>debug</param-name>  
           <param-value>0</param-value>  
       </init-param>  
       <init-param>  
           <param-name>listings</param-name>  
           <param-value>true</param-value>  
       </init-param>  
       <load-on-startup>1</load-on-startup>  
   </servlet>  
```
这里将listings改为true，这个参数控制当访问的资源是物理目录时，是否列出该目录下的文件，为了安全默认是禁止的。

可见，如果在浏览器中输入的资源是一个目录，那么就会列出该目录下的文件，这样是不安全的，所以默认是禁止的。其代码在org.apache.catalina.servlets.DefaultServlet中实现：
```java
if (resource.isDirectory()) {  
           if (!path.endsWith("/")) {  
               doDirectoryRedirect(request, response);  
               return;  
           }  
  
           // Skip directory listings if we have been configured to  
           // suppress them  
           if (!listings) {  
               response.sendError(HttpServletResponse.SC_NOT_FOUND,  
                                  request.getRequestURI());  
               return;  
           }  
           contentType = "text/html;charset=UTF-8";  
       } else {  
           if (!isError) {  
               if (useAcceptRanges) {  
                   // Accept ranges header  
                   response.setHeader("Accept-Ranges", "bytes");  
               }  
  
               // Parse range specifier  
               ranges = parseRange(request, response, resource);  
  
               // ETag header  
               response.setHeader("ETag", eTag);  
  
               // Last-Modified header  
               response.setHeader("Last-Modified", lastModifiedHttp);  
           }  
  
           // Get content length  
           contentLength = resource.getContentLength();  
           // Special case for zero length files, which would cause a  
           // (silent) ISE when setting the output buffer size  
           if (contentLength == 0L) {  
               serveContent = false;  
           }  
       }  
```
当访问的资源是一个物理目录时，会根据listings的值来判断是否列出目录下的文件列表。

## end
