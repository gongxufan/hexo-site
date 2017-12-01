---
layout: post
title: " Java实现转发基于soap协议的webservice接口调用"
date: 2017-03-08 09:36
tags: [java]
category: 工作日志
description: 系统Ａ和系统Ｂ需要交互，但是他们的网络是隔离的。系统Ｂ通过webservice发布了一些接口服务，然后提供WSDL给系统Ａ调用，然而系统A是无法访问系统B的网络，所以数据先要发送中间系统M。
---
## 背景
现在有这么一个需求，我们的系统(M)作为各个系统信息交换中间平台。系统Ａ和系统Ｂ需要交互，但是他们的网络是隔离的。系统Ｂ通过webservice发布了一些接口服务，然后提供WSDL给系统Ａ调用，然而系统A是无法访问系统B的网络，所以数据先要发送中间系统M。其实也就是一个简单的服务转发代理的功能，只是涉及到soap协议的交互。
## 一种简单的方式

系统Ａ使用soapui导入WSDL文件，获取每个方法的请求报文->提交M->原样提交到系统B－>将系统B的处理结果返回
其中主要涉及到soap协议的一些报文头的设置，下边会有调用代码说明。
### 转发的实现

主要是在中间系统Ｍ注册filter来拦截webservice请求，然后将请求数据流构造一个新的http请求发送给系统B。这里请求的发送和转换采用httpcomponents实现。引入依赖：

```xml
<dependency>  
       <groupId>org.apache.httpcomponents</groupId>  
       <artifactId>httpclient</artifactId>  
       <version>4.4</version>  
   </dependency>  
   <dependency>  
       <groupId>org.apache.httpcomponents</groupId>  
       <artifactId>httpcore</artifactId>  
       <version>4.4</version>  
   </dependency>  
<dependency>  
       <groupId>org.apache.httpcomponents</groupId>  
       <artifactId>httpclient</artifactId>  
       <version>4.4</version>  
   </dependency>  
   <dependency>  
       <groupId>org.apache.httpcomponents</groupId>  
       <artifactId>httpcore</artifactId>  
       <version>4.4</version>  
   </dependency> 
```
filter代码如下：
```java
package cn.com.egova.easyshare.web.filter;  
  
  
/** 
 * @auth gongxufan 
 * @Date 2017/9/20 
 **/  
  
  
import org.apache.cxf.helpers.IOUtils;  
import org.apache.http.Header;  
import org.apache.http.client.methods.CloseableHttpResponse;  
import org.apache.http.client.methods.HttpPost;  
import org.apache.http.entity.InputStreamEntity;  
import org.apache.http.impl.client.CloseableHttpClient;  
import org.apache.http.impl.client.HttpClients;  
  
  
import javax.servlet.Filter;  
import javax.servlet.FilterChain;  
import javax.servlet.FilterConfig;  
import javax.servlet.ServletException;  
import javax.servlet.ServletRequest;  
import javax.servlet.ServletResponse;  
import javax.servlet.http.HttpServletRequest;  
import javax.servlet.http.HttpServletResponse;  
import java.io.IOException;  
import java.util.Enumeration;  
import java.util.ResourceBundle;  
  
  
/** 
 * WSDL服务器请求转发 
 */  
public class WebServiceDispatchFilter implements Filter {  
    private static String WEBSERVICE_URL;  
    private static String WEBSERVICE_HOST;  
  
  
    static {  
        ResourceBundle bundler = ResourceBundle.getBundle("config");  
        // 获取调用接口的路径  
        WEBSERVICE_URL = bundler.getString("WEBSERVICE_URL");  
        WEBSERVICE_HOST = bundler.getString("WEBSERVICE_HOST");  
    }  
  
  
    @Override  
    public void doFilter(ServletRequest req, ServletResponse resp,  
                         FilterChain chain) throws IOException, ServletException {  
        HttpServletRequest request = (HttpServletRequest) req;  
        HttpServletResponse response = (HttpServletResponse) resp;  
        try {  
            CloseableHttpClient httpClient = HttpClients.createDefault();  
            HttpPost httpPost = new HttpPost(WEBSERVICE_URL);  
            //构造新的请求数据流  
            InputStreamEntity entity = new InputStreamEntity(  
                    request.getInputStream());  
            httpPost.setEntity(entity);  
            //复制请求参数  
            for (Enumeration<String> e = request.getHeaderNames(); e  
                    .hasMoreElements(); ) {  
                String name = e.nextElement().toString();  
                //注意这里不要设置，发送请求会抛异常  
                if ("Content-Length".equalsIgnoreCase(name)) {  
                    continue;  
                }  
                httpPost.setHeader(name, request.getHeader(name));  
            }  
            //host参数需要覆盖，设置为webservice服务提供这的主机  
            httpPost.setHeader("host", WEBSERVICE_HOST);  
  
  
            //提交soap请求  
            CloseableHttpResponse httpResponse = httpClient.execute(httpPost);  
            //复制soap相应的消息头  
            for (Header h : httpResponse.getAllHeaders()) {  
                response.setHeader(h.getName(), h.getValue());  
            }  
            //讲soap的返回流复制到filter的输出流  
            IOUtils.copyAndCloseInput(httpResponse.getEntity().getContent(),  
                    response.getOutputStream());  
        } catch (Exception e) {  
            response.setContentType("text/xml;charset=UTF-8");  
            String result = "<soap:Envelope xmlns:soap=\"http://schemas.xmlsoap.org/soap/envelope/\">"  
                    + "<soap:Body><soap:Fault><faultcode>"  
                    + "9999"  
                    + "</faultcode>"  
                    + "<faultstring>"  
                    + "失败:"  
                    + e.getMessage()  
                    + "</faultstring></soap:Fault></soap:Body></soap:Envelope>";  
            response.getWriter().print(result);  
        }  
    }  
  
  
    @Override  
    public void init(FilterConfig filterConfig) throws ServletException {  
    }  
  
  
    @Override  
    public void destroy() {  
    }  
} 
```
代码实现也很简单：
```java
CloseableHttpClient httpClient = HttpClients.createDefault();
HttpPost httpPost = new HttpPost(WEBSERVICE_URL);
```
这里构造了一个新的httpPost对象，然后读取请求流构造InputStreamEntity，接着设置httpPost的header。执行CloseableHttpResponse httpResponse = httpClient.execute(httpPost);这样就获取了接口调用的结果最后将httpResponse原样输出到系统A的调用者。

为了测试方便，我们使用天气预报的webservice接口进行测试，上面代码的主机和服务地址是写在配置文件的，其内容如下：

>WEBSERVICE_HOST=www.webxml.com.cn
 WEBSERVICE_URL=http://www.webxml.com.cn/WebServices/WeatherWebService.asmx?wsdl
 
配置filter拦截：
```xml
<filter>  
    <filter-name>WsTransFilter</filter-name>  
    <filter-class>cn.com.egova.easyshare.web.filter.WebServiceDispatchFilter</filter-class>  
</filter>  
<filter-mapping>  
    <filter-name>WsTransFilter</filter-name>  
    <url-pattern>/webservice/*</url-pattern>  
</filter-mapping>  
```

### 使用soapUI进行接口测试
用浏览器访问http://www.webxml.com.cn/WebServices/WeatherWebService.asmx?wsdl，并保存为WeatherWebService.xml。文件内容较多这里就不展示了。

打开soapUI，新建一个soap工程，并导入wsdl文件，这样就可以测试接口了.我们测试下getSupportCity这个方法,只要把地址栏的地址替换成filter的访问地址即可:http://localhost:8080/easyshare-web/
![img](/upload/images/webservice-dispatch/soapui-test.png)

### 使用HttpClient测试接口

只要按照soapUI测试接口的头部信息发送请求报文，就可以成功调用接口。因此我们的客户端调用代码按照这个模式就可以执行接口的调用了，只是要注意我们调用的目标是转发服务的地址。
```java
package cn.com.egova.easyshare.test.web;  
  
import cn.com.egova.easyshare.web.utils.HttpUtils;  
  
import java.util.ResourceBundle;  
  
/** 
 * @auth gongxufan 
 * @Date 2017/9/27 
 **/  
public class SoapInvokeTest {  
    public static void main(String[] arg) {  
        //soap测试  
        String postUrl = "http://localhost:8080/easyshare-web/webservice";  
        //采用SOAP1.1调用服务端，这种方式能调用服务端为soap1.1和soap1.2的服务  
        String orderSoapXml = "<soapenv:Envelope xmlns:soapenv=\"http://schemas.xmlsoap.org/soap/envelope/\" xmlns:web=\"http://WebXml.com.cn/\">\n" +  
                "   <soapenv:Header/>\n" +  
                "   <soapenv:Body>\n" +  
                "      <web:getSupportCity>\n" +  
                "         <!--Optional:-->\n" +  
                "         <web:byProvinceName>云南</web:byProvinceName>\n" +  
                "      </web:getSupportCity>\n" +  
                "   </soapenv:Body>\n" +  
                "</soapenv:Envelope>";  
        String soap1_1 = HttpUtils.doPostSoap1_1(postUrl, orderSoapXml, "http://WebXml.com.cn/getSupportCity");  
        System.out.println(soap1_1);  
        //soap1.2没有soapAction头，直接设置在content-type  
        String soap1_2 = HttpUtils.doPostSoap1_2(postUrl, orderSoapXml, "http://WebXml.com.cn/getSupportCity");  
        System.out.println(soap1_2);  
        System.out.println("end");  
  
    }  
}  
```

```java
public static String doPostSoap1_1(String postUrl, String soapXml,  
                                      String soapAction) {  
       LOGGER.info("开始转发webservice：" + postUrl);  
       String retStr = "";  
       // 创建HttpClientBuilder  
       HttpClientBuilder httpClientBuilder = HttpClientBuilder.create();  
       // HttpClient  
       CloseableHttpClient closeableHttpClient = httpClientBuilder.build();  
       HttpPost httpPost = new HttpPost(postUrl);  
       //  设置请求和传输超时时间  
      /* RequestConfig requestConfig = RequestConfig.custom() 
               .setSocketTimeout(socketTimeout) 
               .setConnectTimeout(connectTimeout).build(); 
       httpPost.setConfig(requestConfig);*/  
       try {  
           httpPost.setHeader("Content-Type", "text/xml;charset=UTF-8");  
           httpPost.setHeader("SOAPAction", soapAction);  
           StringEntity data = new StringEntity(soapXml,  
                   Charset.forName("UTF-8"));  
           httpPost.setEntity(data);  
           CloseableHttpResponse response = closeableHttpClient  
                   .execute(httpPost);  
           HttpEntity httpEntity = response.getEntity();  
           if (httpEntity != null) {  
               // 打印响应内容  
               retStr = EntityUtils.toString(httpEntity, "UTF-8");  
               LOGGER.info("response:" + retStr);  
           }  
  
       } catch (Exception e) {  
           LOGGER.error("exception in doPostSoap1_1", e);  
       } finally {  
           // 释放资源  
           try {  
               closeableHttpClient.close();  
           } catch (IOException e) {  
               e.printStackTrace();  
           }  
       }  
       return retStr;  
   }  
  
   public static String doPostSoap1_2(String postUrl, String soapXml,  
                                      String soapAction) {  
       String retStr = "";  
       // 创建HttpClientBuilder  
       HttpClientBuilder httpClientBuilder = HttpClientBuilder.create();  
       // HttpClient  
       CloseableHttpClient closeableHttpClient = httpClientBuilder.build();  
       HttpPost httpPost = new HttpPost(postUrl);  
     /*  // 设置请求和传输超时时间 
       RequestConfig requestConfig = RequestConfig.custom() 
               .setSocketTimeout(socketTimeout) 
               .setConnectTimeout(connectTimeout).build(); 
       httpPost.setConfig(requestConfig);*/  
       try {  
           httpPost.setHeader("Content-Type",  
                   "text/xml;charset=UTF-8;action=\"" + soapAction + "\"");  
           StringEntity data = new StringEntity(soapXml,  
                   Charset.forName("UTF-8"));  
           httpPost.setEntity(data);  
           CloseableHttpResponse response = closeableHttpClient  
                   .execute(httpPost);  
           HttpEntity httpEntity = response.getEntity();  
           if (httpEntity != null) {  
               // 打印响应内容  
               retStr = EntityUtils.toString(httpEntity, "UTF-8");  
               LOGGER.info("response:" + retStr);  
           }  
           // 释放资源  
           closeableHttpClient.close();  
       } catch (Exception e) {  
           LOGGER.error("exception in doPostSoap1_2", e);  
       }  
       return retStr;  
   }
```
上面1.1和1.2方法的区别就是头部设置的差异，这个是按照soapUI调用时自动生成的header来设置的。

![img](/upload/images/webservice-dispatch/http-test.png)

## end

本文实现的方法就是按照原始的soapUI调用，主要导入WSDL文件根据其定义的方法，将请求报文和请求头发送到代理服务器然后在转发到webservice服务方即可实现调用的转发。

其实我们还可以采用apache camel进行服务的转发，不过camel适用于较复杂的规则路由场景。其具体使用也很方便，只要配置好路由规则即可实现的服务的路由和转发，具体可参考其文档。
