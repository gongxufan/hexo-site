---
layout: post
title: "基于springmvc+spring-data-jpa+dubbo开发web应用"
date: 2017-05-12 11:23
tags: [spring,java]
category: 随笔
description: 集成springMVC、spring-data-jpa,dubbo进行web开发。

---
最近有项目客户要求使用dubbo进行服务关联和分布式部署，能基于dubbo发布分布式的rest服务和基于wsdl的webservice服务。看了下dubbo这个项目，其开发基本处于停滞状态,比较活跃的项目只有当当网维护的dubbox项目。本项目就是基于dubbox进行的。

项目模块如下图所示
![img](/upload/images/springmvc-dubbo/dubbo-project.png)

用pom来组织Jar包依赖，dubbox我已经打包成Jar。可以看到项目分为四个模块。api提供rest和webservice的接口暴露，common是一些实体和工具类,server是数据库操作的模块，这里我们使用spring-data-jpa进行操作。web是前端应用，它通过api模块暴露的接口来调用远程dubbo服务，包括rest和webservice的。
## api部分的实现
### 1、暴露rest接口
```java
package cn.com.egova.easyshare.api.human;  
  
import cn.com.egova.easyshare.common.dto.ResultDto;  
import cn.com.egova.easyshare.common.entity.Human;  
import com.alibaba.dubbo.rpc.protocol.rest.support.ContentType;  
  
import javax.ws.rs.Consumes;  
import javax.ws.rs.GET;  
import javax.ws.rs.Path;  
import javax.ws.rs.PathParam;  
import javax.ws.rs.Produces;  
import javax.ws.rs.core.MediaType;  
  
/** 
 * 人员相关接口 
 * Created by gongxufan on 2017/3/8. 
 */  
@Path("humans")  
@Consumes({MediaType.APPLICATION_JSON, MediaType.TEXT_XML})  
@Produces({ContentType.APPLICATION_JSON_UTF_8, ContentType.TEXT_XML_UTF_8})  
public interface HumanActionApi<T> {  
  
    /** 
     * 获取人员信息 
     * 
     * @param id 
     * @return 
     */  
    @GET  
    @Path("view/{id : \\d+}")  
    ResultDto<Human> getHuman(@PathParam("id") Integer id);  
}  
```
这里使用rest的注解暴露接口
### 2、暴露webservice
```java
public interface WSDLServiceApi {  
    List<NewsDto> fetchNews(@XmlJavaTypeAdapter(StringObjectMapAdapter.class) Map params);  
}  
```
webservice的写法很简单，只是需要注意要在接口返回的实体类上上加上xml的注解
```java
package cn.com.egova.easyshare.common.dto;  
  
import javax.xml.bind.annotation.XmlAccessType;  
import javax.xml.bind.annotation.XmlAccessorType;  
import javax.xml.bind.annotation.XmlElement;  
import javax.xml.bind.annotation.XmlRootElement;  
import java.util.Date;  
  
/** 
 * Created by gongxufan on 2017/3/30. 
 */  
@XmlRootElement  
@XmlAccessorType(XmlAccessType.FIELD)  
public class NewsDto {  
  @XmlElement  
  private Integer newsID;  
  @XmlElement  
  private String newsName;  
  @XmlElement  
  private String newsDescr;  
  @XmlElement  
  private String newsURL;  
  @XmlElement  
  private Integer newsTypeID;  
  @XmlElement  
  private Integer unitID;  
  @XmlElement  
  private Date createDate;  
  @XmlElement  
  private Date updateDate;  
  @XmlElement  
  private Integer displayOrder;  
  @XmlElement  
  private Integer openFlag = 0;  
  @XmlElement  
  private String op;  
  
  public Integer getNewsID() {  
    return newsID;  
  }  
  
  public void setNewsID(Integer newsID) {  
    this.newsID = newsID;  
  }  
  
  public String getNewsName() {  
    return newsName;  
  }  
  
  public void setNewsName(String newsName) {  
    this.newsName = newsName;  
  }  
  
  public String getNewsDescr() {  
    return newsDescr;  
  }  
  
  public void setNewsDescr(String newsDescr) {  
    this.newsDescr = newsDescr;  
  }  
  
  public String getNewsURL() {  
    return newsURL;  
  }  
  
  public void setNewsURL(String newsURL) {  
    this.newsURL = newsURL;  
  }  
  
  public Integer getNewsTypeID() {  
    return newsTypeID;  
  }  
  
  public void setNewsTypeID(Integer newsTypeID) {  
    this.newsTypeID = newsTypeID;  
  }  
  
  public Integer getUnitID() {  
    return unitID;  
  }  
  
  public void setUnitID(Integer unitID) {  
    this.unitID = unitID;  
  }  
  
  public Date getCreateDate() {  
    return createDate;  
  }  
  
  public void setCreateDate(Date createDate) {  
    this.createDate = createDate;  
  }  
  
  public Date getUpdateDate() {  
    return updateDate;  
  }  
  
  public void setUpdateDate(Date updateDate) {  
    this.updateDate = updateDate;  
  }  
  
  public Integer getDisplayOrder() {  
    return displayOrder;  
  }  
  
  public void setDisplayOrder(Integer displayOrder) {  
    this.displayOrder = displayOrder;  
  }  
  
  public Integer getOpenFlag() {  
    return openFlag;  
  }  
  
  public void setOpenFlag(Integer openFlag) {  
    this.openFlag = openFlag;  
  }  
  
  public String getOp() {  
    return op;  
  }  
  
  public void setOp(String op) {  
    this.op = op;  
  }  
}  
```
api模块单独分出来可以直接提供给其他模块进行调用
## 接口实现部分

### 使用dubbo服务
在app-config.xml中我们需要定义dubbo服务相关的信息，具体网上很多资料可以查看。
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<!--  
 - Copyright 1999-2011 Alibaba Group.  
 -    
 - Licensed under the Apache License, Version 2.0 (the "License");  
 - you may not use this file except in compliance with the License.  
 - You may obtain a copy of the License at  
 -    
 -      http://www.apache.org/licenses/LICENSE-2.0  
 -    
 - Unless required by applicable law or agreed to in writing, software  
 - distributed under the License is distributed on an "AS IS" BASIS,  
 - WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.  
 - See the License for the specific language governing permissions and  
 - limitations under the License.  
-->  
<beans xmlns="http://www.springframework.org/schema/beans"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
    xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"  
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd  
    http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">  
  
    <dubbo:application name="service-provider" owner="egova" organization="goutu"/>  
  
    <dubbo:registry address="zookeeper://127.0.0.1:2181" check="false"/>  
  
    <!--使Dubbo在Spring容器初始化完后，再暴露服务-->  
    <dubbo:provider delay="-1"/>  
    <!--uncomment this if you want to test dubbo's monitor-->  
    <!--<dubbo:monitor protocol="registry"/>-->  
    <!-- use tomcat server -->  
    <dubbo:protocol name="rest" port="8888" threads="500" contextpath="services" server="tomcat" accepts="1000"  
    extension="cn.com.egova.easyshare.api.extension.CustomExceptionMapper"/>  
  
    <dubbo:protocol name="webservice" port="8892" server="tomcat" contextpath="services"/>  
</beans>  
```
这里我们定义了rest和webservice协议，rest协议在8888下而webservice在8892下访问。需要注意的是，我们把这两个服务都跑在tomcat下。这样我们的接口实现部分可以打成war包部署在多个tomcat下，从而实现了接口服务的分布式部署。

这里只是作为演示，所以app-db.xml中的数据源定义都已经注释掉了。如果你想采用spring-data-jpa进行数据操作的话可以把注释去掉，然后去实现自己的Repository 。
## web应用部分
这个就不用提了，我提供了一个测试页面测试接口。需要注意的是，web调用接口的配置是在dubbo-consumer.xml中
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
    xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"  
    xsi:schemaLocation="http://www.springframework.org/schema/beans  
        http://www.springframework.org/schema/beans/spring-beans.xsd  
        http://code.alibabatech.com/schema/dubbo  
        http://code.alibabatech.com/schema/dubbo/dubbo.xsd">  
    <!-- 消费方应用名，用于计算依赖关系，不是匹配条件，不要与提供方一样 -->  
    <dubbo:application name="dubbo-client" owner="egova-client" organization="goutu-client" />  
    <!-- zookeeper注册中心 -->  
    <dubbo:registry  protocol="zookeeper" address="127.0.0.1:2181" check="false"/>  
    <!-- 监控中心配置 -->  
    <!--<dubbo:monitor protocol="registry"/>-->  
  
    <!--关闭所有服务的启动可用性检查-->  
    <dubbo:consumer check="false" />  
  
     <!-- dubbo远程服务-->  
    <dubbo:reference id="humanAction"   interface="cn.com.egova.easyshare.api.human.HumanActionApi"/>  
    <dubbo:reference id="wSDLService"   interface="cn.com.egova.easyshare.webservice.WSDLServiceApi" />  
</beans>
```  
在这里配置好zookeeper和api端的service,然后在contoller里就可以调用接口方法了。

## end
代码下载地址:[csdn下载](http://download.csdn.net/download/qq381332153/9840564)
github: [https://github.com/gongxufan/dubbo-demo](https://github.com/gongxufan/dubbo-demo)