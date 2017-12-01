---
layout: post
title: "基于spring-boot&spring-data-jpa的web开发环境集成"
date: 2016-11-14 18:21
tags: [spring,java]
category: 工作日志
description: 搭建基于spring-boot和spring-data-jpa进行开发的项目骨架。
---
## 新技术？
spring-boot并不是全新的技术栈，而是整合了spring的很多组件，并且以约定优先的原则进行组合。使用boot我们不需要对冗杂的配置文件进行管理，主需要用它的注解便可启用大部分web开发中所需要的功能。本篇就是基于boot来配置jpa和静态文件访问，进行web应用的开发。
## 模板or静态页面

最原始的jsp页面在springboot中已经不在默认支持，spring-boot默认使用thymeleaf最为模板。当然我们也可以使用freemark或者velocity等其他后端模板。但是按照前后端分离的良好设计，我们最好采用静态页面作为前端模板，这样前后端完全分离，把数据处理逻辑写在程序并提供接口供前端调用。这样的设计更加灵活和清晰。

## 项目搭建
我们将讨论项目的结构、application配置文件、静态页面处理、自定义filter，listener,servlet以及拦截器的使用。最后集中讨论jpa的配置和操作以及如何进行单元测试和打包部署。
### 项目结构
项目使用maven进行依赖管理和构建，整体结构如下图所示：
![img](/upload/images/spring/spring-data-jpa-dir.png)
我们的HTML页面和资源文件都在resources/static下，打成jar包的时候static目录位于/BOOT-INF/classes/。
#### pom.xml
我们需要依赖下面这些包：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>gxf.dev</groupId>
    <artifactId>topology</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.6.RELEASE</version>
    </parent>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.apache.tomcat</groupId>
                    <artifactId>tomcat-jdbc</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

      <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
        <dependency>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
        </dependency>

        <!--test-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-test</artifactId>
            <version>4.3.10.RELEASE</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <repositories>
        <repository>
            <id>nexus-aliyun</id>
            <name>Nexus aliyun</name>
            <layout>default</layout>
            <url>http://maven.aliyun.com/nexus/content/groups/public</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
            <releases>
                <enabled>true</enabled>
            </releases>
        </repository>
    </repositories>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <mainClass>gxf.dev.topology.Application</mainClass>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```
spring-boot-starter-parent使我们项目的父pom。
spring-boot-starter-web提供嵌入式tomcat容器，从而使项目可以通过打成jar包的方式直接运行。
spring-boot-starter-data-jpa引入了jpa的支持。
spring-boot-test和junit配合做单元测试。
mysql-connector-java和HikariCP做数据库的连接池的操作。
spring-boot-maven-plugin插件能把项目和依赖的jar包以及资源文件和页面一起打成一个可运行的jar（运行在内嵌的tomcat）

#### 启动人口
```java
package gxf.dev.topology;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.servlet.ServletComponentScan;

@SpringBootApplication
@EnableAutoConfiguration
@ServletComponentScan
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class);
    }
}
```
这里ServletComponentScan注解是启用servlet3的servler和filter以及listener的支持，下面会提到该用法。要注意的是：**不能引入@EnableWebMvc注解**，否则需要重新配置视图和资源文件映射。这样就不符合我们的前后端分离的初衷了。
###  静态资源处理
spring-boot默认会去classpath下面的/static/，/public/ ，/resources/目录去读取静态资源。因此按照约定优先的原则，我们直接把我们应用的页面和资源文件直接放在/static下面，如下图所示：
![img](/upload/images/spring/spring-boot-dir.png)
这样我们访问系统主页就会自动加载index.html，而且它所引用的资源文件也会在static/下开始加载。
### application.yml
我们在application配置文件中设置各种参数，它可以是传统的properties文件也可以使用yml来逐级配置。本文采用的第二种方式yml，如果不懂可以参考：[https://baike.baidu.com/item/YAML/1067697?fr=aladdin](https://baike.baidu.com/item/YAML/1067697?fr=aladdin)。其内容如下：
```xml
server:
    port: 8080
    context-path: /topology
    session:
      timeout: 30
    tomcat:
      uri-encoding: utf-8

logging:
    level:
        root: info
        gxf.dev.topology: debug
        #当配置了loggin.path属性时，将在该路径下生成spring.log文件，即：此时使用默认的日志文件名spring.log
        #当配置了loggin.file属性时，将在指定路径下生成指定名称的日志文件。默认为项目相对路径，可以为logging.file指定绝对路径。
        #path: /home/gongxufan/logs
    file: topology.log

spring:
    jpa:
      show-sql: true
      open-in-view: false
      hibernate:
        naming:
          #配置ddl建表字段和实体字段一致
          physical-strategy: gxf.dev.topology.config.RealNamingStrategyImpl
          ddl-auto: update
      properties:
        hibernate:
          format_sql: true
          show_sql: true
          dialect: org.hibernate.dialect.MySQL5Dialect
    datasource:
        url: jdbc:mysql://localhost:3306/topology
        driver-class-name: com.mysql.jdbc.Driver
        username: root
        password: qwe
        hikari:
              cachePrepStmts: true
              prepStmtCacheSize: 250
              prepStmtCacheSqlLimit: 2048
              useServerPrepStmts: true

```
使用idea开发工具在编辑器会有自动变量提示，这样非常方便进行参数的选择和查阅。
#### server
server节点可以配置容器的很多参数，比如：端口，访问路径、还有tomcat本身的一些参数。这里设置了session的超时以及编码等。
#### logging
日志级别可以定义到具体的哪个包路径，日志文件的配置要注意：path和file配置一个就行，file默认会在程序工作目录下生成，也可以置顶绝对路径进行指定。
#### datasource
这里使用号称性能最牛逼的连接池hikaricp，具体配置可以参阅其官网：[http://brettwooldridge.github.io/HikariCP/](http://brettwooldridge.github.io/HikariCP/)
#### jpa
这里主要注意下strategy的配置，关系到自动建表时的字段命名规则。默认会生成带_划线分割entity的字段名(骆驼峰式)。
```java
package gxf.dev.topology.config;

/**
 * ddl-auto选项开启的时候生成表的字段命名策略，默认会按照骆驼峰式风格用_隔开每个单词
 * 这个类可以保证entity定义的字段名和数据表的字段一致
 * @auth gongxufan
 * @Date 2016/8/3
 **/

import org.hibernate.boot.model.naming.Identifier;
import org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl;
import org.hibernate.engine.jdbc.env.spi.JdbcEnvironment;

import java.io.Serializable;


public class RealNamingStrategyImpl extends org.springframework.boot.orm.jpa.hibernate.SpringPhysicalNamingStrategy implements Serializable {

    public static final PhysicalNamingStrategyStandardImpl INSTANCE = new PhysicalNamingStrategyStandardImpl();

    @Override
    public Identifier toPhysicalTableName(Identifier name, JdbcEnvironment context) {
        return new Identifier(name.getText(), name.isQuoted());
    }

    @Override
    public Identifier toPhysicalColumnName(Identifier name, JdbcEnvironment context) {
        return new Identifier(name.getText(), name.isQuoted());
    }

}
```
### 注册web组件
1) 最新的spring-boot引入新的注解ServletComponentScan，使用它可以方便的配置Servlet3+的web组件。主要有下面这三个注解：
```java
@WebServlet
@WebFilter
@WebListener
```
只要把这些注解标记组件即可完成注册。
```java
package gxf.dev.topology.filter;

import org.springframework.core.annotation.Order;

import javax.servlet.Filter;
import javax.servlet.FilterChain;
import javax.servlet.FilterConfig;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.annotation.WebFilter;
import java.io.IOException;

/**
 * author:gongxufan
 * date:11/14/17
 **/
@Order(1)
@WebFilter(filterName = "loginFilter", urlPatterns = "/login")
public class LoginFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("login rquest");
        chain.doFilter(request,response);
    }

    @Override
    public void destroy() {

    }
}

```
```java
package gxf.dev.topology.filter;

import javax.servlet.annotation.WebListener;
import javax.servlet.http.HttpSessionAttributeListener;
import javax.servlet.http.HttpSessionBindingEvent;
import javax.servlet.http.HttpSessionEvent;
import javax.servlet.http.HttpSessionListener;

/**
 * 自定义listener
 * Created by gongxufan on 2016/7/5.
 */
@WebListener
public class SessionListener implements HttpSessionListener,HttpSessionAttributeListener {

    @Override
    public void sessionCreated(HttpSessionEvent httpSessionEvent) {
        System.out.println("init");
    }

    @Override
    public void sessionDestroyed(HttpSessionEvent httpSessionEvent) {
        System.out.println("destroy");
    }

    @Override
    public void attributeAdded(HttpSessionBindingEvent se) {
        System.out.println(se.getName() + ":" + se.getValue());
    }

    @Override
    public void attributeRemoved(HttpSessionBindingEvent se) {

    }

    @Override
    public void attributeReplaced(HttpSessionBindingEvent se) {

    }
}

```
2) 拦截器的使用
拦截器不是Servlet规范的标准组件，它跟上面的三个组件不在一个处理链上。拦截器是spring使用AOP实现的，对controller执行前后可以进行干预，直接结束请求处理。而且拦截器只能对流经dispatcherServlet处理的请求才生效，静态资源就不会被拦截。
下面顶一个拦截器：
```java
package gxf.dev.topology.filter;

import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

/**
 * author:gongxufan
 * date:11/14/17
 **/
public class LoginInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o) throws Exception {
        System.out.println("LoginInterceptor.preHandle()在请求处理之前进行调用（Controller方法调用之前）");
        // 只有返回true才会继续向下执行，返回false取消当前请求
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, ModelAndView modelAndView) throws Exception {
        System.out.println("LoginInterceptor.postHandle()请求处理之后进行调用，但是在视图被渲染之前（Controller方法调用之后）");
    }

    @Override
    public void afterCompletion(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception e) throws Exception {
        System.out.println("LoginInterceptor.afterCompletion()在整个请求结束之后被调用，也就是在DispatcherServlet 渲染了对应的视图之后执行（主要是用于进行资源清理工作）");
    }
}

```
要想它生效则需要加入拦截器栈：
```java
package gxf.dev.topology.config;

import gxf.dev.topology.filter.LoginInterceptor;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;

/**
 * author:gongxufan
 * date:11/14/17
 **/
@Configuration
public class WebMvcConfigurer extends WebMvcConfigurerAdapter {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
    	//在这可以配置controller的访问路径
        registry.addInterceptor(new LoginInterceptor()).addPathPatterns("/**");
        super.addInterceptors(registry);
    }
}

```
### jpa操作
spring-boot已经集成了JPA的Repository封装，基于注解的事务处理等，我们只要按照常规的JPA使用方法即可。以Node表的操作为例：
1. 定义entity
```java
package gxf.dev.topology.entity;

import com.fasterxml.jackson.annotation.JsonInclude;

import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.Table;
import java.io.Serializable;

/**
 * Created by gongxufan on 2014/11/20.
 */
@Entity
@Table(name = "node")
@JsonInclude(JsonInclude.Include.NON_NULL)
public class Node implements Serializable {

    @Id
    private String id;
    private String elementType;
    private String x;
    private String y;
    private String width;
    private String height;
    private String alpha;
    private String rotate;
    private String scaleX;
    private String scaleY;
    private String strokeColor;
    private String fillColor;
    private String shadowColor;
    private String shadowOffsetX;
    private String shadowOffsetY;
    private String zIndex;
    private String text;
    private String font;
    private String fontColor;
    private String textPosition;
    private String textOffsetX;
    private String textOffsetY;
    private String borderRadius;
    private String deviceId;
    private String dataType;
    private String borderColor;
    private String offsetGap;
    private String childNodes;
    private String nodeImage;
    private String templateId;
    private String deviceA;
    private String deviceZ;
    private String lineType;
    private String direction;
    private String vmInstanceId;
    private String displayName;
    private String vmid;
    private String topoLevel;
    private String parentLevel;
    private Setring nextLevel;
    //getter&setter
}
```
JsonInclude注解用于返回JOSN字符串是忽略为空的字段。
2. 编写repository接口
```java
package gxf.dev.topology.repository;

import gxf.dev.topology.entity.Node;
import org.springframework.data.repository.PagingAndSortingRepository;

public interface NodeRepository extends PagingAndSortingRepository<Node, String> {
}

```
3. 编写Service
```java
package gxf.dev.topology.service;

import gxf.dev.topology.entity.Node;
import gxf.dev.topology.repository.NodeRepository;
import gxf.dev.topology.repository.SceneRepository;
import gxf.dev.topology.repository.StageRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

/**
 * dao操作
 * author:gongxufan
 * date:11/13/17
 **/
@Component
public class TopologyService {

    @Autowired
    private NodeRepository nodeRepository;

    @Autowired
    private SceneRepository sceneRepository;

    @Autowired
    private StageRepository stageRepository;

    @Transactional
    public Node saveNode(Node node) {
        return nodeRepository.save(node);
    }

    public Iterable<Node> getAll() {
        return nodeRepository.findAll();
    }
}

```
### 单元测试
单元测试使用spring-boot-test和junit进行，需要用到下面的几个注解：
```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
```
测试代码如下：
```java
import gxf.dev.topology.Application;
import gxf.dev.topology.entity.Node;
import gxf.dev.topology.repository.CustomSqlDao;
import gxf.dev.topology.service.TopologyService;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

/**
 * author:gongxufan
 * date:11/13/17
 **/
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
public class ServiceTest {
    @Autowired
    private TopologyService topologyService;

    @Autowired
    private CustomSqlDao customSqlDao;
    @Test
    public void testNode() {
        Node node = new Node();
        node.setId("node:2");
        node.setDisplayName("test1");
        topologyService.saveNode(node);
    }

    @Test
    public void testNative(){
        System.out.println(customSqlDao.querySqlObjects("select * from node"));
        System.out.println(customSqlDao.getMaxColumn("id","node"));
    }
}

```
### jpa补充
使用JPA进行单表操作确实很方便，但是对于多表连接的复杂查询可能不太方便。一般有两种方式弥补这个不足：
1. 一个是在Query里标注为NativeQuery，直接使用原生SQL。不过这种方式在动态参数查询到额情况下很不方便，这时候我们需要按条件拼接SQL。
2. 自定义DAO
```java
package gxf.dev.topology.repository;

import com.mysql.jdbc.StringUtils;
import org.hibernate.SQLQuery;
import org.hibernate.criterion.CriteriaSpecification;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import javax.persistence.EntityManager;
import javax.persistence.EntityManagerFactory;
import javax.persistence.Query;
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * 支持自定义SQL查询
 * Created by gongxufan on 2016/3/17.
 */
@Component
public class CustomSqlDao {
    @Autowired
    private EntityManagerFactory entityManagerFactory;

    public int getMaxColumn(final String filedName, final String tableName) {
        String sql = "select nvl(max(" + filedName + "), 0)  as max_num from " + tableName;
        Map map =  entityManagerFactory.getProperties();
        String dialect = (String) map.get("hibernate.dialect");
        //determine which database use
        if(!StringUtils.isNullOrEmpty(dialect)){
            if(dialect.contains("MySQL")){
                sql = "select ifnull(max(" + filedName + "), 0)  as max_num from " + tableName;
            }
            if(dialect.contains("Oracle")){
                sql = "select nvl(max(" + filedName + "), 0)  as max_num from " + tableName;
            }
        }
        int maxID = 0;
        List<Map<String, Object>> list = this.querySqlObjects(sql);
        if (list.size() > 0) {
            Object maxNum = list.get(0).get("max_num");
            if(maxNum instanceof Number)
                maxID = ((Number)maxNum).intValue();
            if(maxNum instanceof String)
                maxID = Integer.valueOf((String)maxNum);
        }
        return maxID + 1;
    }

    public List<Map<String, Object>> querySqlObjects(String sql, Integer currentPage, Integer rowsInPage) {
        return this.querySqlObjects(sql, null, currentPage, rowsInPage);
    }

    public List<Map<String, Object>> querySqlObjects(String sql) {
        return this.querySqlObjects(sql, null, null, null);
    }

    public List<Map<String, Object>> querySqlObjects(String sql, Map params) {
        return this.querySqlObjects(sql, params, null, null);
    }

    @SuppressWarnings("unchecked")
    public List<Map<String, Object>> querySqlObjects(String sql, Object params, Integer currentPage, Integer rowsInPage) {
        EntityManager entityManager = entityManagerFactory.createEntityManager();
        Query qry = entityManager.createNativeQuery(sql);
        SQLQuery s = qry.unwrap(SQLQuery.class);

        //设置参数
        if (params != null) {
            if (params instanceof List) {
                List<Object> paramList = (List<Object>) params;
                for (int i = 0, size = paramList.size(); i < size; i++) {
                    qry.setParameter(i + 1, paramList.get(i));
                }
            } else if (params instanceof Map) {
                Map<String, Object> paramMap = (Map<String, Object>) params;
                Object o = null;
                for (String key : paramMap.keySet()) {
                    o = paramMap.get(key);
                    if (o != null)
                        qry.setParameter(key, o);
                }
            }
        }

        if (currentPage != null && rowsInPage != null) {//判断是否有分页
            // 起始对象位置
            qry.setFirstResult(rowsInPage * (currentPage - 1));
            // 查询对象个数
            qry.setMaxResults(rowsInPage);
        }
        s.setResultTransformer(CriteriaSpecification.ALIAS_TO_ENTITY_MAP);
        List<Map<String, Object>> resultList = new ArrayList<Map<String, Object>>();
        try {
            List list = qry.getResultList();
            resultList = s.list();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }
        return resultList;
    }


    public int getCount(String sql) {
        String sqlCount = "select count(0) as count_num from " + sql;
        List<Map<String, Object>> list = this.querySqlObjects(sqlCount);
        if (list.size() > 0) {
            int countNum = ((BigDecimal) list.get(0).get("COUNT_NUM")).intValue();
            return countNum;
        } else {
            return 0;
        }
    }

    /**
     * 处理sql语句
     *
     * @param _strSql
     * @return
     */
    public String toSql(String _strSql) {
        String strNewSql = _strSql;

        if (strNewSql != null) {
            strNewSql = regReplace("'", "''", strNewSql);
        } else {
            strNewSql = "";
        }

        return strNewSql;
    }

    private String regReplace(String strFind, String strReplacement, String strOld) {
        String strNew = strOld;
        Pattern p = null;
        Matcher m = null;
        try {
            p = Pattern.compile(strFind);
            m = p.matcher(strOld);
            strNew = m.replaceAll(strReplacement);
        } catch (Exception e) {
        }

        return strNew;
    }

    /**
     * 根据hql语句查询数据
     *
     * @param hql
     * @return
     */
    @SuppressWarnings("rawtypes")
    public List queryForList(String hql, List<Object> params) {
        EntityManager entityManager = entityManagerFactory.createEntityManager();
        Query query = entityManager.createQuery(hql);
        List list = null;
        try {
            if (params != null && !params.isEmpty()) {
                for (int i = 0, size = params.size(); i < size; i++) {
                    query.setParameter(i + 1, params.get(i));
                }
            }
            list = query.getResultList();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }
        return list;
    }

    @SuppressWarnings("rawtypes")
    public List queryByMapParams(String hql, Map<String, Object> params, Integer currentPage, Integer pageSize) {
        EntityManager entityManager = entityManagerFactory.createEntityManager();
        Query query = entityManager.createQuery(hql);
        List list = null;
        try {
            if (params != null && !params.isEmpty()) {
                for (Map.Entry<String, Object> entry : params.entrySet()) {
                    query.setParameter(entry.getKey(), entry.getValue());
                }
            }

            if (currentPage != null && pageSize != null) {
                query.setFirstResult((currentPage - 1) * pageSize);
                query.setMaxResults(pageSize);
            }
            list = query.getResultList();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }

        return list;
    }

    @SuppressWarnings("rawtypes")
    public List queryByMapParams(String hql, Map<String, Object> params) {
        return queryByMapParams(hql, params, null, null);
    }

    @SuppressWarnings("rawtypes")
    public List queryForList(String hql) {
        return queryForList(hql, null);
    }


    /**
     * 查询总数
     *
     * @param hql
     * @param params
     * @return
     */
    public Long queryCount(String hql, Map<String, Object> params) {
        EntityManager entityManager = entityManagerFactory.createEntityManager();
        Query query = entityManager.createQuery(hql);
        Long count = null;
        try {
            if (params != null && !params.isEmpty()) {
                for (Map.Entry<String, Object> entry : params.entrySet()) {
                    query.setParameter(entry.getKey(), entry.getValue());
                }
            }
            count = (Long) query.getSingleResult();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            entityManager.close();
        }

        return count;
    }

    /**
     * 查询总数
     *
     * @param sql
     * @param params
     * @return
     */
    public Integer queryCountBySql(String sql, Map<String, Object> params) {
        EntityManager entityManager = entityManagerFactory.createEntityManager();
        Integer count = null;
        try {
            Query query = entityManager.createNativeQuery(sql);
            if (params != null && !params.isEmpty()) {
                for (Map.Entry<String, Object> entry : params.entrySet()) {
                    query.setParameter(entry.getKey(), entry.getValue());
                }
            }

            Object obj = query.getSingleResult();
            if (obj instanceof BigDecimal) {
                count = ((BigDecimal) obj).intValue();
            } else {
                count = (Integer) obj;
            }

        } finally {
            if (entityManager != null) {
                entityManager.close();
            }
        }
        return count;
    }

    /**
     * select count(*) from table
     *
     * @param sql
     * @param params
     * @return
     */
    public int executeSql(String sql, List<Object> params) {
        EntityManager entityManager = entityManagerFactory.createEntityManager();
        try {
            Query query = entityManager.createNativeQuery(sql);
            if (params != null && !params.isEmpty()) {
                for (int i = 0, size = params.size(); i < size; i++) {
                    query.setParameter(i + 1, params.get(i));
                }
            }
            return query.executeUpdate();
        } finally {
            if (entityManager != null) {
                entityManager.close();
            }
        }
    }
}

```
我们在service层注入，然后就可以根据输入条件拼接好sql或者hql来进行各种操作。这种方式灵活而且也不需要手动写分页代码，使用hibernate封装好的机制即可。
## 总结
使用boot可以快速搭建一个前后端开发的骨架，里面有很多自动的配置和约定。虽然boot不是新的一个技术栈，但是它要求我们对各个组件都要比较熟悉，不然对它的运行机制和约定配置会感到很困惑。而使用JPA进行数据库操作也是利弊参半，需要自己权衡。

项目代码：[https://github.com/gongxufan/topology](https://github.com/gongxufan/topology)


