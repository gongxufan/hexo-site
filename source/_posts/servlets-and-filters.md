---
layout: post
title: "Probably the largest collection of Java(tm) Servlets and filters"
date: 2017-10-20 16:45
tags: java
category: 转载
description: 海量的web开发可以用上的过滤器和小程序以及各种工具。

---
如题所述“Probably the largest collection of Java(tm) Servlets and filters”，该站点提供了安全，session,IP,时间等工具。其提供的每一个工具包都有简要的介绍和使用说明，如果我们想进一步了解其实现细节或者二次开发可以反编译查看源码。

比如我们常用的XSS过滤：

This is a Java servlet filter (as per Servlet API 2.3). This filter lets you deal with Cross Site Scripting (XSS) attempts. Filter intercepts every request sent to your web application and then cleans any potential script injection. What it basically does is remove all suspicious strings from request parameters (and headers) before returning them to the application.
How to use it:
- download xssflt.jar and save it in WEB-INF/lib
- describe this filter in web.xml. An optional initial parameter apostrophe lets you define the replacement code for apostrophe (') sign. By default it is &#39;. 
```xml
<filter> 
  <filter-name>XSSFilter</filter-name> 
  <filter-class>com.cj.xss.XSSFilter</filter-class> 
</filter>
```
- describe a mapping for this filter in web.xml. E.g.: 
```xml
<filter-mapping> 
  <filter-name>XSSFilter</filter-name> 
  <url-pattern>/*</url-pattern> 
</filter-mapping>
```
in this example filter will be on for the each file.

这样的使用说明真是大道至简。。。

## 下载地址
servletsuite站点地址：[http://www.servletsuite.com/filters.htm](http://www.servletsuite.com/filters.htm)
csdn本地全集下载地址：[http://download.csdn.net/download/qq381332153/10026088](http://download.csdn.net/download/qq381332153/10026088)
