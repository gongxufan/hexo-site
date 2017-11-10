---
layout: post
title: "微信.公众平台开发模式配置管理系统"
date: 2017-11-06 14:46
tags: [java,前端]
category: 工作日志
description: 仿微信官方公众号后台管理系统,实现公众号的菜单管理、用户管理、消息管理、素材管理、消息群发，自动回复等基本功能。
top: 998
---

## 前言
偶然发现磁盘中还有一个早期的关于微信后台管理的项目，闲来无事便在此廖记。

这个项目是使用SpringMvc+sqllite实现的仿微信官方公众号平台管理系统，实现了菜单管理、用户管理、消息管理、素材管理、消息群发，自动回复等基本功能，集成了ueditor和微信sdk，采用基于注解的SpringMVC模式开发：

## 主要功能实现

配置注解的包扫描路径：
```java
package cn.com.egova.wx.config.web;  
  
import org.springframework.context.annotation.ComponentScan;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.context.annotation.FilterType;  
import org.springframework.web.servlet.config.annotation.EnableWebMvc;  
/** 
 ** Created by gongxufan on 2016/7/30. 
 */  
@Configuration  
@ComponentScan(basePackages = {"cn.com.egova.wx"},  
        excludeFilters = {  
                @ComponentScan.Filter(type = FilterType.ANNOTATION, value = EnableWebMvc.class)  
        })  
public class RootConfig {  
}  
```

基于WebMvcConfigureAdatper配置视图等web配置：
```java
package cn.com.egova.wx.config.web;  
  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.ComponentScan;  
import org.springframework.context.annotation.Configuration;  
import org.springframework.web.servlet.ViewResolver;  
import org.springframework.web.servlet.config.annotation.DefaultServletHandlerConfigurer;  
import org.springframework.web.servlet.config.annotation.EnableWebMvc;  
import org.springframework.web.servlet.config.annotation.WebMvcConfigurerAdapter;  
import org.springframework.web.servlet.view.InternalResourceViewResolver;  
  
/** 
 ** SpringMVC组件配置，包括视图解析器、静态资源访问处理等等 
 ** <p> 
 ** Created by gongxufan on 2016/7/30. 
 */  
@Configuration  
@EnableWebMvc  
@ComponentScan(basePackages = {"cn.com.egova.wx.web.controller", "cn.com.egova.wx.web.service"})  
public class WebConfig extends WebMvcConfigurerAdapter {  
    @Bean  
    public ViewResolver viewResolver() {  
        //定义标准的jsp视图解析器  
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();  
        resolver.setPrefix("/WEB-INF/jsp/");  
        resolver.setSuffix(".jsp");  
        resolver.setExposeContextBeansAsAttributes(true);  
        return resolver;  
    }  
    @Override  
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {  
        //静态资源访问交由容易默认的servlet处理  
        configurer.enable();  
    }  
}  
```

配置过滤器：
```java
import com.jfinal.core.JFinalFilter;  
import org.springframework.web.filter.CharacterEncodingFilter;  
import org.springframework.web.servlet.support.AbstractAnnotationConfigDispatcherServletInitializer;  
import org.springframework.web.util.Log4jConfigListener;  
  
import javax.servlet.FilterRegistration;  
import javax.servlet.ServletContext;  
import javax.servlet.ServletException;  
import java.util.HashMap;  
import java.util.Map;  
  
/** 
 ** web.xml的编码方式，各种组件（监听器，过滤器等）在此初始化和注册 
 ** 注解方式的配置，需求容器支持Servlet3.0规范 
 ** Created by gongxufan on 2016/7/30. 
 */  
public class WxSetupWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {  
  
    @Override  
    public void onStartup(ServletContext servletContext) throws ServletException {  
        //配置全局参数  
        servletContext.setInitParameter("log4jConfigLocation", "/WEB-INF/log4j.xml");  
        servletContext.setInitParameter("webAppRootKey", "com.egova.wx");  
  
        //添加编码过滤器  
        FilterRegistration.Dynamic encodingFilter = servletContext.addFilter("characterEncodingFilter",  
                CharacterEncodingFilter.class);  
        encodingFilter.addMappingForUrlPatterns(null, false, "/*");  
        Map<String, String> params = new HashMap<String, String>();  
        params.put("encoding", "UTF-8");  
        params.put("forceEncoding", "true");  
        encodingFilter.setInitParameters(params);  
  
        //启动JFinal微信  
        FilterRegistration.Dynamic wxFilter = servletContext.addFilter("handler", JFinalFilter.class);  
        wxFilter.addMappingForUrlPatterns(null, false, "/weixin/*");  
        params = new HashMap<String, String>();  
        params.put("configClass", "cn.com.egova.wx.config.wx.WeixinConfig");  
        wxFilter.setInitParameters(params);  
  
        //log4j监听器  
        servletContext.addListener(Log4jConfigListener.class);  
  
        super.onStartup(servletContext);  
    }  
  
    @Override  
    protected Class<?>[] getRootConfigClasses() {  
        return new Class<?>[]{RootConfig.class};  
    }  
  
    @Override  
    protected Class<?>[] getServletConfigClasses() {  
        return new Class<?>[]{WebConfig.class};  
    }  
  
    @Override  
    protected String[] getServletMappings() {  
        //DispatcherServlet映射到/请求地址  
        return new String[]{"/"};  
    }  
} 
```

## 集成微信SDK

通过添加过滤器的方式集成jfinal的微信sdk,具体在
```java
 //启动JFinal微信
FilterRegistration.Dynamic wxFilter = servletContext.addFilter("handler", JFinalFilter.class);
wxFilter.addMappingForUrlPatterns(null, false, "/weixin/*");
params = new HashMap<String, String>();
params.put("configClass", "cn.com.egova.wx.config.wx.WeixinConfig");
wxFilter.setInitParameters(params);
```
此处对/weixin/开头的请求进行拦截，并配置了参数类WeixinConfig,其代码如下：
```java
package cn.com.egova.wx.config.wx;

/**
 * 微信公众号相关配置
 * Created by gongxufan on 2016/8/2.
 */

import cn.com.egova.wx.web.handler.WeixinApiInvokeHandler;
import cn.com.egova.wx.web.handler.WeixinMsgMonitorHandler;
import cn.com.egova.wx.sdk.api.ApiConfigKit;
import com.jfinal.config.Constants;
import com.jfinal.config.Handlers;
import com.jfinal.config.Interceptors;
import com.jfinal.config.JFinalConfig;
import com.jfinal.config.Plugins;
import com.jfinal.config.Routes;
import com.jfinal.kit.PropKit;
import com.jfinal.render.ViewType;

public class WeixinConfig extends JFinalConfig {
    /**
     * 如果生产环境配置文件存在，则优先加载该配置，否则加载开发环境配置文件
     *
     * @param pro 生产环境配置文件
     * @param dev 开发环境配置文件
     */
    public void loadProp(String pro, String dev) {
        try {
            PropKit.use(pro);
        } catch (Exception e) {
            PropKit.use(dev);
        }
    }

    @Override
    public void configConstant(Constants me) {
        loadProp("/sysinfo.properties", "/sysinfo.properties");
        me.setDevMode(PropKit.getBoolean("devMode", false));
        me.setEncoding("utf-8");
        me.setViewType(ViewType.JSP);
        me.setError500View("/500.html");
        me.setError404View("/404.html");
        // ApiConfigKit 设为开发模式可以在开发阶段输出请求交互的 xml 与 json 数据
        ApiConfigKit.setDevMode(me.getDevMode());
    }

    @Override
    public void configRoute(Routes me) {
        me.add("/weixin/msg", WeixinMsgMonitorHandler.class,"/");
        me.add("/weixin/api", WeixinApiInvokeHandler.class,"/");
    }

    @Override
    public void configPlugin(Plugins me) {
    }

    @Override
    public void configInterceptor(Interceptors me) {
    }

    @Override
    public void configHandler(Handlers me) {
    }
}

```
在这里配置msg和api的处理器，分别是处理微信消息和微信api调用的处理逻辑

## 系统主要功能界面
![img](/upload/images/weixin-manager/main-func.png)
自定义菜单还是实现了跟官方一样的菜单排序，菜单的的图文消息编辑等
![img](/upload/images/weixin-manager/menu-manage.png)
素材管理实现图文消息的编辑和新建，图片的管理和查看
![img](/upload/images/weixin-manager/muti-message.png)
![img](/upload/images/weixin-manager/muti-message-mg.png)
可以设置自动回复和消息群发
![img](/upload/images/weixin-manager/auto-reply.png)
其他功能就不在此赘述，附上项目地址：https://github.com/gongxufan/wechat-conf/

## end