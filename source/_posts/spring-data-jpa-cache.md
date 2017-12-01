---
layout: post
title: "spring-data-jpa使用缓存的注意事项"
date: 2016-07-20 14:14
tags: [java,spring] 
category: 随笔
description: 使用spring-data-jpa的查询缓存和二级缓存的注意事项。

---
## 前言
采用hibernate的JPA实现，对于简单的查询十分方便。而对于复杂查询我们也可以写SQL来进行复杂的多表连接查询。很多人不喜欢hibernate其实更多的是对其机制的掌握不深，如果认真研究其实现源码，其实是一个很快乐的学习过程。各种设计范式的运用也是精彩绝伦。

这里主要说下缓存的配置。既然是hibernate，其缓存机制离不开这三种：session级别的缓存、sessionFactory级别的二级缓存以及查询缓存和带有集合的缓存。对于jpa的使用，个人建议不要使用关系映射。这样会导致各种关联查询，涉及到延迟加载等机制，会消耗额外的cglib的大量代理工作。尤其是很多人还使用org.springframework.orm.jpa.support.OpenEntityManagerInViewFilter来实现集合的页面懒加载。这对http响应要求很高的系统简直就是灾难。

使用jpa要放弃hibernate的关系映射部分，保持开发的简洁和灵活性。

先上测试代码
```java
package cn.com.egova.easyshare.test.service;  
  
import cn.com.egova.easyshare.common.dto.NavMenuDto;  
import cn.com.egova.easyshare.common.entity.Human;  
import cn.com.egova.easyshare.common.entity.NavMenu;  
import cn.com.egova.easyshare.server.service.IHumanService;  
import org.junit.Test;  
import org.junit.runner.RunWith;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.test.context.ContextConfiguration;  
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;  
  
import javax.persistence.EntityManager;  
import javax.persistence.EntityManagerFactory;  
import java.util.List;  
  
@RunWith(SpringJUnit4ClassRunner.class)  
@ContextConfiguration({"classpath:/applicationContext.xml"})  
public class HumanServiceTest {  
    @Autowired  
    private IHumanService humanService;  
  
    @Autowired  
    private EntityManagerFactory entityManagerFactory;  
  
    //一级缓存：内建的session级别的缓存,默认开启  
    @Test  
    public void sessionCacheTest(){  
        EntityManager entityManager = entityManagerFactory.createEntityManager();  
  
        Human human = entityManager.find(Human.class,37);  
        Human human1 = entityManager.find(Human.class,37);  
        System.out.println(human == human1?"true":"false");  
        entityManager.close();  
  
        entityManager = entityManagerFactory.createEntityManager();  
  
        Human human3 = entityManager.find(Human.class,37);  
        System.out.println(human == human3?"true":"false");  
    }  
  
    //二级缓存，同一个sessionFactory的不同session缓存，默认关闭  
    public void sessionFactoryCacheTest(){  
        EntityManager entityManager = entityManagerFactory.createEntityManager();  
  
        NavMenu navMenu = entityManager.find(NavMenu.class,1);  
        entityManager.close();  
  
        entityManager = entityManagerFactory.createEntityManager();  
        NavMenu navMenu2 = entityManager.find(NavMenu.class,1);  
        entityManager.close();  
  
        System.out.println(navMenu == navMenu2?"true":"false");  
    }  
  
    //查询缓存，前两种都是根据主键ID的缓存策略.  
    @Test  
    public void queryCacheTest() throws Exception{  
        List<NavMenuDto> navMenuDtos = humanService.getMenuList(37);  
        List<NavMenuDto> navMenuDtos2 = humanService.getMenuList(37);  
    }  
  
}  
```
## 一级缓存
也就是自建的session级别的缓存，在一个session回话事物中根据主键查询是只会查询一次数据库的。
```java
EntityManager entityManager = entityManagerFactory.createEntityManager();  
Human human = entityManager.find(Human.class,37);  
Human human1 = entityManager.find(Human.class,37);  
System.out.println(human == human1?"true":"false");  
entityManager.close();  
entityManager = entityManagerFactory.createEntityManager();  
Human human3 = entityManager.find(Human.class,37);  
 System.out.println(human == human3?"true":"false"); 
```
控制台打印：
```sql
Hibernate:   
    select  
        human0_.humanID as humanID1_11_0_,  
        human0_.activeFlag as activeFlag2_11_0_,  
        human0_.address as address3_11_0_,  
        human0_.createDate as createDate4_11_0_,  
        human0_.description as description5_11_0_,  
        human0_.displayOrder as displayOrder6_11_0_,  
        human0_.email as email7_11_0_,  
        human0_.humanCode as humanCode8_11_0_,  
        human0_.humanName as humanName9_11_0_,  
        human0_.humanPassword as humanPassword10_11_0_,  
        human0_.identifyType as identifyType11_11_0_,  
        human0_.orgId as orgId12_11_0_,  
        human0_.tel as tel13_11_0_,  
        human0_.unitID as unitID14_11_0_,  
        human0_.userid as userid15_11_0_,  
        human0_.userroles as userroles16_11_0_,  
        human0_.zipCode as zipCode17_11_0_   
    from  
        tcHuman human0_   
    where  
        human0_.humanID=?  
true  
2016-07-20 14:24:30 INFO main org.hibernate.engine.internal.StatisticalLoggingSessionEventListener - Session Metrics {
    399787 nanoseconds spent acquiring 1 JDBC connections;  
    0 nanoseconds spent releasing 0 JDBC connections;  
    114082560 nanoseconds spent preparing 1 JDBC statements;  
    2907307 nanoseconds spent executing 1 JDBC statements;  
    0 nanoseconds spent executing 0 JDBC batches;  
    0 nanoseconds spent performing 0 L2C puts;  
    0 nanoseconds spent performing 0 L2C hits;  
    0 nanoseconds spent performing 0 L2C misses;  
    0 nanoseconds spent executing 0 flushes (flushing a total of 0 entities and 0 collections);  
    0 nanoseconds spent executing 0 partial-flushes (flushing a total of 0 entities and 0 collections)  
}  
Hibernate:   
    select  
        human0_.humanID as humanID1_11_0_,  
        human0_.activeFlag as activeFlag2_11_0_,  
        human0_.address as address3_11_0_,  
        human0_.createDate as createDate4_11_0_,  
        human0_.description as description5_11_0_,  
        human0_.displayOrder as displayOrder6_11_0_,  
        human0_.email as email7_11_0_,  
        human0_.humanCode as humanCode8_11_0_,  
        human0_.humanName as humanName9_11_0_,  
        human0_.humanPassword as humanPassword10_11_0_,  
        human0_.identifyType as identifyType11_11_0_,  
        human0_.orgId as orgId12_11_0_,  
        human0_.tel as tel13_11_0_,  
        human0_.unitID as unitID14_11_0_,  
        human0_.userid as userid15_11_0_,  
        human0_.userroles as userroles16_11_0_,  
        human0_.zipCode as zipCode17_11_0_   
    from  
        tcHuman human0_   
    where  
        human0_.humanID=?  
false  
```
执行`entityManager.close(); `entityManager关闭后会重新执行一次sql查询

## 二级缓存
二级缓存默认是关闭的，需要配置缓存策略
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
       xmlns:context="http://www.springframework.org/schema/context"  
       xmlns:aop="http://www.springframework.org/schema/aop"  
       xmlns:tx="http://www.springframework.org/schema/tx"  
       xmlns:p="http://www.springframework.org/schema/p"  
       xmlns:cache="http://www.springframework.org/schema/cache"  
       xmlns:jpa="http://www.springframework.org/schema/data/jpa"  
  
       xsi:schemaLocation="http://www.springframework.org/schema/beans  
         http://www.springframework.org/schema/beans/spring-beans-2.5.xsd  
          http://www.springframework.org/schema/context  
          http://www.springframework.org/schema/context/spring-context-3.1.xsd  
          http://www.springframework.org/schema/aop  
          http://www.springframework.org/schema/aop/spring-aop-3.1.xsd  
          http://www.springframework.org/schema/tx  
          http://www.springframework.org/schema/tx/spring-tx-3.1.xsd  
          http://www.springframework.org/schema/cache  
          http://www.springframework.org/schema/cache/spring-cache-3.1.xsd  
          http://www.springframework.org/schema/data/jpa  
          http://www.springframework.org/schema/data/jpa/spring-jpa.xsd">  
  
    <context:annotation-config />  
  
    <context:component-scan base-package="cn.com.egova.easyshare"/>  
    <!--配置文件读取-->  
    <bean id="propertyConfigurer" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">  
        <property name="locations">  
            <list>  
                <value>classpath:config.properties</value>  
            </list>  
        </property>  
  
    </bean>  
    <bean id="springContextUtil" class="cn.com.egova.easyshare.common.utils.SpringContextUtil"></bean>  
    <bean id="dataSource" class="org.apache.commons.dbcp2.BasicDataSource">  
        <property name="driverClassName" value="${db.driverClassName}"></property>  
        <property name="url" value="${db.url}"></property>  
        <property name="username" value="${db.username}"></property>  
        <property name="password" value="${db.password}"></property>  
        <property name="maxTotal" value="${db.maxTotal}"></property>  
        <property name="maxIdle" value="${db.maxIdle}"></property>  
    </bean>  
  
    <bean id="entityManagerFactory"  
          class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">  
        <property name="dataSource" ref="dataSource" />  
        <property name="jpaVendorAdapter">  
            <bean class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter">  
                <property name="generateDdl" value="false" />  
                <property name="database" value="ORACLE" />  
            </bean>  
        </property>  
        <!-- 指定Entity实体类包路径 -->  
        <property name="packagesToScan">  
            <value>cn.com.egova.easyshare.common.entity</value>  
        </property>  
        <!-- 指定JPA属性；如Hibernate中指定是否显示SQL的是否显示、方言等 -->  
        <property name="jpaProperties">  
            <props>  
                <prop key="hibernate.dialect">org.hibernate.dialect.Oracle9iDialect</prop>  
                <prop key="hibernate.show_sql">true</prop>  
                <prop key="hibernate.format_sql">true</prop>  
                <prop key="hibernate.generate_statistics">true</prop>  
                <prop key="hibernate.hbm2ddl.auto">none</prop>  
                <!-- 配置二级缓存 -->  
                <prop key="hibernate.cache.use_second_level_cache">true</prop>  
                <prop key="hibernate.cache.region.factory_class">org.hibernate.cache.ehcache.EhCacheRegionFactory</prop>  
                <!-- 开启查询缓存 -->  
                <prop key="hibernate.cache.use_query_cache">true</prop>  
                <prop key="hibernate.cache.provider_configuration">classpath:ehcache.xml</prop>  
            </props>  
        </property>  
    </bean>  
  
    <!-- 配置事务管理器 -->  
    <bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager">  
        <property name="entityManagerFactory" ref="entityManagerFactory" />  
    </bean>  
  
    <!-- 启用 annotation事务-->  
    <tx:annotation-driven transaction-manager="transactionManager"/>  
      
      
    <!-- 采用JdbcTemplate配置 -->  
    <bean id="jdbcTemplate"  class="org.springframework.jdbc.core.JdbcTemplate">    
        <property name="dataSource" ref="dataSource" />    
    </bean>  
      
    <bean id="lobHandler"  class="org.springframework.jdbc.support.lob.OracleLobHandler">    
    </bean>  
      
    <!-- 通过aop配置事务 -->  
    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">  
        <property name="dataSource" ref="dataSource"/>  
    </bean>  
      
  
    <aop:aspectj-autoproxy/>  
    <!-- 配置Spring Data JPA扫描目录-->  
    <jpa:repositories base-package="cn.com.egova.easyshare.server.repository"/>  
</beans>  
```
然后在需要使用缓存的entity配置好缓存注解
```java
package cn.com.egova.easyshare.common.entity;  
  
import org.hibernate.annotations.Cache;  
import org.hibernate.annotations.CacheConcurrencyStrategy;  
  
import javax.persistence.*;  
  
/** 
 * 系统菜单 
 * Created by gongxufan on 2016/3/13.
 */  
@Entity  
@Table(name = "TCSYSNAVMENU")  
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)  
@Cacheable(true)  
public class NavMenu {  
    @Id  
    @GeneratedValue(strategy = GenerationType.TABLE, generator = "tableKeyGenerator")  
    @TableGenerator(name = "tableKeyGenerator", table = "tcTableKeyGenerator",  
            pkColumnName = "pk_key", valueColumnName = "pk_value", pkColumnValue = "menuID",  
            initialValue = 1, allocationSize = 1)  
    private Integer menuID;  
    private Integer seniorMenuID;  
    private String menuName;  
    private String menuDescr;  
    private String menuIconClass;  
    private String hrefUrl;  
    private Integer displayOrder;  
  
    public Integer getMenuID() {  
        return menuID;  
    }  
  
    public void setMenuID(Integer menuID) {  
        this.menuID = menuID;  
    }  
  
    public Integer getSeniorMenuID() {  
        return seniorMenuID;  
    }  
  
    public void setSeniorMenuID(Integer seniorMenuID) {  
        this.seniorMenuID = seniorMenuID;  
    }  
  
    public String getMenuName() {  
        return menuName;  
    }  
  
    public void setMenuName(String menuName) {  
        this.menuName = menuName;  
    }  
  
    public String getMenuDescr() {  
        return menuDescr;  
    }  
  
    public void setMenuDescr(String menuDescr) {  
        this.menuDescr = menuDescr;  
    }  
  
    public String getMenuIconClass() {  
        return menuIconClass;  
    }  
  
    public void setMenuIconClass(String menuIconClass) {  
        this.menuIconClass = menuIconClass;  
    }  
  
    public String getHrefUrl() {  
        return hrefUrl;  
    }  
  
    public void setHrefUrl(String hrefUrl) {  
        this.hrefUrl = hrefUrl;  
    }  
  
    public Integer getDisplayOrder() {  
        return displayOrder;  
    }  
  
    public void setDisplayOrder(Integer displayOrder) {  
        this.displayOrder = displayOrder;  
    }  
}  
```
当然还要配置好ehcache.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
         xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd"  
         updateCheck="false">  
  
    <!-- Sets the path to the directory where cache .data files are created.  
  
         If the path is a Java System Property it is replaced by  
         its value in the running VM.  
  
         The following properties are translated:  
         user.home - User's home directory  
         user.dir - User's current working directory  
         java.io.tmpdir - Default temp file path -->  
    <!-- 缓存写入文件目录 -->  
    <diskStore path="user.home"/>  
  
  
    <!--Default Cache configuration. These will applied to caches programmatically created through  
        the CacheManager.  
  
        The following attributes are required for defaultCache:  
  
        maxInMemory       - Sets the maximum number of objects that will be created in memory  
        eternal           - Sets whether elements are eternal. If eternal,  timeouts are ignored and the element  
                            is never expired.  
        timeToIdleSeconds - Sets the time to idle for an element before it expires. Is only used  
                            if the element is not eternal. Idle time is now - last accessed time  
        timeToLiveSeconds - Sets the time to live for an element before it expires. Is only used  
                            if the element is not eternal. TTL is now - creation time  
        overflowToDisk    - Sets whether elements can overflow to disk when the in-memory cache  
                            has reached the maxInMemory limit.  
  
        -->  
    <!-- 数据过期策略 -->  
    <defaultCache  
            maxElementsInMemory="10000"  
            eternal="false"  
            timeToIdleSeconds="120"  
            timeToLiveSeconds="120"  
            overflowToDisk="true"  
    />  
  
    <!--Predefined caches.  Add your cache configuration settings here.  
        If you do not have a configuration for your cache a WARNING will be issued when the  
        CacheManager starts  
  
        The following attributes are required for defaultCache:  
  
        name              - Sets the name of the cache. This is used to identify the cache. It must be unique.  
        maxInMemory       - Sets the maximum number of objects that will be created in memory  
        eternal           - Sets whether elements are eternal. If eternal,  timeouts are ignored and the element  
                            is never expired.  
        timeToIdleSeconds - Sets the time to idle for an element before it expires. Is only used  
                            if the element is not eternal. Idle time is now - last accessed time  
        timeToLiveSeconds - Sets the time to live for an element before it expires. Is only used  
                            if the element is not eternal. TTL is now - creation time  
        overflowToDisk    - Sets whether elements can overflow to disk when the in-memory cache  
                            has reached the maxInMemory limit.  
  
        -->  
  
    <!-- Sample cache named sampleCache1  
        This cache contains a maximum in memory of 10000 elements, and will expire  
        an element if it is idle for more than 5 minutes and lives for more than  
        10 minutes.  
  
        If there are more than 10000 elements it will overflow to the  
        disk cache, which in this configuration will go to wherever java.io.tmp is  
        defined on your system. On a standard Linux system this will be /tmp"  
        -->  
  
    <!-- Sample cache named sampleCache2  
        This cache contains 1000 elements. Elements will always be held in memory.  
        They are not expired. -->  
    <!-- <cache name="sampleCache2"  
        maxElementsInMemory="1000"  
        eternal="true"  
        timeToIdleSeconds="0"  
        timeToLiveSeconds="0"  
        overflowToDisk="false"  
        />  -->  
  
    <!-- Place configuration for your caches following -->  
  
</ehcache>  
```
运行测试代码会发现两次执行只有一次sql查询

## 查询缓存
前两种都是对主键查询时候的缓存，对于普通的sql查询我们可以在repository中配置hints
```java
package cn.com.egova.easyshare.server.repository;  
  
import cn.com.egova.easyshare.common.entity.Human;  
import cn.com.egova.easyshare.common.entity.NavMenu;  
import org.springframework.data.jpa.repository.Query;  
import org.springframework.data.jpa.repository.QueryHints;  
import org.springframework.data.repository.PagingAndSortingRepository;  
import org.springframework.data.repository.query.Param;  
  
import javax.persistence.QueryHint;  
import java.util.List;  
  
/** 
 * Created by gongxufan on 2016/3/13.
 */  
public interface SysNavRepository extends PagingAndSortingRepository<NavMenu, Integer> {  
  
    @Query("select n from NavMenu n where exists (select 1 from SysNavRight s " +  
            "where n.menuID=s.sysNavMenuID and s.humanID=:humanId) order by n.displayOrder")  
    @QueryHints({ @QueryHint(name = "org.hibernate.cacheable", value ="true") })  
    List<NavMenu> findMenuList(@Param("humanId") Integer humanId);  
  
    @Query("select m from NavMenu  m  order by m.displayOrder")  
    List<NavMenu> findAllMenuList();  
}  
```
运行测试代码
```java
    @Test  
   public void queryCacheTest() throws Exception{  
       List<NavMenuDto> navMenuDtos = humanService.getMenuList(37);  
       List<NavMenuDto> navMenuDtos2 = humanService.getMenuList(37);  
   }
```
控制台输出
```sql
Hibernate:   
    select  
        navmenu0_.menuID as menuID1_4_,  
        navmenu0_.displayOrder as displayOrder2_4_,  
        navmenu0_.hrefUrl as hrefUrl3_4_,  
        navmenu0_.menuDescr as menuDescr4_4_,  
        navmenu0_.menuIconClass as menuIconClass5_4_,  
        navmenu0_.menuName as menuName6_4_,  
        navmenu0_.seniorMenuID as seniorMenuID7_4_   
    from  
        TCSYSNAVMENU navmenu0_   
    where  
        exists (  
            select  
                1   
            from  
                TCHUMANSYSRIGHTS sysnavrigh1_   
            where  
                navmenu0_.menuID=sysnavrigh1_.sysNavMenuID   
                and sysnavrigh1_.humanID=?  
        )   
    order by  
        navmenu0_.displayOrder  
2016-07-20 14:32:13 INFO main org.hibernate.engine.internal.StatisticalLoggingSessionEventListener - Session Metrics {
    445013 nanoseconds spent acquiring 1 JDBC connections;  
    0 nanoseconds spent releasing 0 JDBC connections;  
    179244800 nanoseconds spent preparing 1 JDBC statements;  
    15228160 nanoseconds spent executing 1 JDBC statements;  
    0 nanoseconds spent executing 0 JDBC batches;  
    20012371 nanoseconds spent performing 39 L2C puts;  
    0 nanoseconds spent performing 0 L2C hits;  
    145920 nanoseconds spent performing 1 L2C misses;  
    0 nanoseconds spent executing 0 flushes (flushing a total of 0 entities and 0 collections);  
    0 nanoseconds spent executing 0 partial-flushes (flushing a total of 0 entities and 0 collections)  
}  
2016-07-20 14:32:13 INFO main org.hibernate.engine.internal.StatisticalLoggingSessionEventListener - Session Metrics {
    0 nanoseconds spent acquiring 0 JDBC connections;  
    0 nanoseconds spent releasing 0 JDBC connections;  
    0 nanoseconds spent preparing 0 JDBC statements;  
    0 nanoseconds spent executing 0 JDBC statements;  
    0 nanoseconds spent executing 0 JDBC batches;  
    0 nanoseconds spent performing 0 L2C puts;  
    1620055 nanoseconds spent performing 39 L2C hits;  
    36694 nanoseconds spent performing 2 L2C misses;  
    0 nanoseconds spent executing 0 flushes (flushing a total of 0 entities and 0 collections);  
    0 nanoseconds spent executing 0 partial-flushes (flushing a total of 0 entities and 0 collections)  
}  

```
这可以看到第二次执行同样的查询会有L2C缓存命中

## 总结
如何配置二级缓存和注意事项
1. 首先需要添加pom依赖
```xml
<dependency>  
    <groupId>org.springframework.data</groupId>  
    <artifactId>spring-data-jpa</artifactId>  
    <version>${spring-data-jpa.version}</version>  
</dependency>  
<dependency>  
    <groupId>org.hibernate</groupId>  
    <artifactId>hibernate-core</artifactId>  
    <version>${hibernate.version}</version>  
</dependency>  
<dependency>  
    <groupId>org.hibernate</groupId>  
    <artifactId>hibernate-ehcache</artifactId>  
    <version>${hibernate.version}</version>  
</dependency>  

```
一定要注意ehcache和hibernate的版本要一致，不然启动会报错。

2. 修改jpa配置
```xml
<prop key="hibernate.generate_statistics">true</prop>  
<!-- 配置二级缓存 -->  
<prop key="hibernate.cache.use_second_level_cache">true</prop>  
<prop key="hibernate.cache.region.factory_class">org.hibernate.cache.ehcache.EhCacheRegionFactory</prop>  
<!-- 开启查询缓存 -->  
<prop key="hibernate.cache.use_query_cache">true</prop>  
<prop key="hibernate.cache.provider_configuration">classpath:ehcache.xml</prop>  
```
这里开启了二级缓存和查询缓存，指定了ehcache缓存的配置文件statistics是个调试开关，可以查询缓存的命中情况
3. ehcache.xml配置
```xml
<defaultCache  
           maxElementsInMemory="10000"  
           eternal="false"  
           timeToIdleSeconds="120"  
           timeToLiveSeconds="120"  
           overflowToDisk="true"  
   />  
```
`timeToIdleSeconds`  
这个是缓存的最大空闲时间，也就是多久不访问自动失效
`timeToLiveSeconds`
这个是最大存活时间，如果在存活时间内空闲时间达到上限，缓存也会自动失效。
4)给entity配置缓存策略
```java
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)  
@Cacheable(true)  
```
cacheable主要对集合类的缓存提供支持，缓存策略这里适合数据更新不多的情况进行设置。

## end
使用二级缓存和查询缓存要注意使用场景，如果发现增加缓存机制系统反而出现吞吐下降频发宕机。就要考虑是不是该换memcache等专门的缓存服务器了。