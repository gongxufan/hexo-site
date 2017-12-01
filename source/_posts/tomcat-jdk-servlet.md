---
layout: post
title: "tomcat和servlet以及jdk版本的对应关系"
date: 2016-11-07 14:28
tags: [tomcat]
category: 随笔
description: 介绍servlet、tomcat和jdk的版本关系。
---
最新的servlet规范是servlet4.0，然而很多项目在编译的时候用的servlet2.5，导致迁移到不同版本的tomcat启动发生错误。由于servlet3.0以后启用了nio以及对filter和servlet的注解支持，性能和使用上都得到了极大的提升。如果不是要兼容很老的系统，实在没必要使用2.5版本。

下面这个表格列出了所有tomcat和servlet版本以及jdk的对应关系：
![img](/upload/images/tomcat/tomcat-version.png)
每个版本的Tomcat都支持满足上表中最后一列标明的java稳定版，可以在任何符合上表中最后一列要求的Java早期访问构建(比如Java9正式发布之前，tomcat9就可以在其早起构建版本上运行)上工作。但是不排除有bug和问题。

因此我们应该按照我们开发使用的jdk版本和tomcat版本对照，选择合适的稳定版本用于生产：

>Alpha版本可能包含大量未测试的/缺少规范和/或重大bug所需要的功能，不能指望它能稳定运行。
Beta版本可能包含一些未测试的功能和/或一些相对较小的bug,Beta版本不保证稳定运行。
Stable版本可能包含少量相对较小的bug，稳定的发行版是用于生产的，在很长一段时间内都能稳定运行。

从上面的表可以看出：从tomcat7开始支持servlet3.0规范，而如果我们使用Java８进行开发最好使用8.5.23版本(稳定版),8.0.x是废弃的主线版本。

在servlet2.5中`request.getParameterMap()`方法返回`Map<String, String>`，而3.0以后返回的是`Map<String, String[]>`，这样升级会导致一些兼容性问题。

注意
另外使用JSP2.0的话JSTL标签应该写成下面的方式：
```jsp
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>  
<%@ taglib prefix="fmt" uri="http://java.sun.com/jsp/jstl/fmt" %>  
```
写成下面的方式是会报错的：
```jsp
<%@ taglib uri="http://java.sun.com/jstl/core" prefix="c" %>  
<%@ taglib prefix="fmt" uri="http://java.sun.com/jstl/fmt" %> 
```
这样会出现下面的错误信息：
> According to TLD or attribute directive in tag file, attribute [value] does not accept any expressions

也可以使用上边写法的_rt版本：
```jsp
<%@ taglib uri="http://java.sun.com/jstl/core_rt" prefix="c" %>  
<%@ taglib prefix="fmt" uri="http://java.sun.com/jstl/fmt_rt" %>  
```
tomcat9.x是现在的主要开发版本，它是基于8.0.x和8.5x进行开发的。主要是实现servlet4.0规范，JSP2.3和EL3.0以及WebSocket1.1。另外9.x版本也添加了对Java9的支持，包括HTTP2.0。
最后在maven中引用如下：
```xml
<dependency>  
      <groupId>javax.servlet</groupId>  
      <artifactId>javax.servlet-api</artifactId>  
      <version>3.1.0</version>  
      <scope>provided</scope>  
  </dependency>  
```