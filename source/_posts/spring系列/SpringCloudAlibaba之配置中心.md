---
title: spring cloud alibaba系列(三)Nacos Config配置中心
date: 2019-01-29 16:10:45
tags: 
- springcloud 
- alibaba
categories: springcloud

---



## 概述
- Nacos 是阿里巴巴开源的一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。
- Nacos Config就是一个类似于SpringCloud Config的配置中心

## 接入
- SpringCloud项目集成Nacos Config配置中心很简单。只需要部署Nacos 客户端并在里面添加配置即可。然后引入Nacos Config动态读取即可

 **1. 创建一个SpringCloud工程cloud-config 修改 pom.xml 文件，引入 Nacos Config Starter** 

前提得选引入spring-cloud-alibaba-dependencies
```java
 <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
 </dependency>
```

 **2. 修改application.properties配置** 

```java
server.port=18085
management.endpoints.web.exposure.include=*
```
 **3. 新建bootstrap.properties文件** 

```java
spring.application.name=cloud-config
spring.cloud.nacos.config.server-addr=127.0.0.1:8848
```

 **4. 启动Nacos客户端** 
* 具体步骤可参考 [上篇文章](https://blog.qinxuewu.club/2019/01/27/spring-xi-lie/springcloudalibaba-zhi-fu-wu-zhu-ce-fa-xian/) 



 **5. 配置列表新增配置** 
* `dataId` ：格式如下 `${prefix} - ${spring.profiles.active} . ${file-extension}`
* prefix 默认为 spring.application.name 的值
* spring.profiles.active 当前环境对应的 profile

* file-extension 为配置内容的数据格式，可以通过配置项 spring.cloud.nacos.config.file-extension来配置。 目前只支持 properties 类型。
* group 默认`DEFAULT_GROUP`

`当activeprofile 为空时直接填写 spring.application.name值即可 默认properties`

![](https://img-blog.csdnimg.cn/20190321095416198.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)

 **5. 添加Controller并在类上添加 @RefreshScope动态获取配置中心的值** 
```java
@RestController
@RefreshScope
public class SampleController {

    @Value("${user.name}")
    String userName;

    @Value("${user.age}")
    int age;


    @RequestMapping("/user")
    public String simple() {
        return "获取 Nacos Config配置如下："  + userName + " " + age + "!";
    }
}
```

 **6.启动测试** 

这样就可以获取到配置中心的值。并且配置中心修改值后。可以立即动态刷新获取最新的值

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190321095451780.png)

* 案例源码: https://github.com/a870439570/alibaba-cloud

![觉得本文不错的话，分享一下给小伙伴吧~](http://wx1.sinaimg.cn/large/006b7Nxngy1g1eu6ewhl9j30760763yz.jpg)