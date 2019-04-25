---
title: Spring Boot Admin2.1应用监控
date: 2018-10-26 13:41:57
tags: springbootAdmin
categories: springboot
---


* Spring Boot Admin 是一个管理和监控Spring Boot 应用程序的开源软件。每个应用都认为是一个客户端，通过HTTP或者使用 Eureka注册到admin server中进行展示，Spring Boot Admin UI部分使用AngularJs将数据展示在前端。
* Spring Boot Admin 是一个针对spring-boot的actuator接口进行UI美化封装的监控工具。他可以：在列表中浏览所有被监控spring-boot项目的基本信息，详细的Health信息、内存信息、JVM信息、垃圾回收信息、各种配置信息（比如数据源、缓存列表和命中率）等，还可以直接修改logger的level。


### 设置Spring Boot Admin Server
- 新建一个springBoot2.x工程，将Spring Boot Admin Server启动器添加到pom.xml
- 使用ide新建工程可以直接选择引入Spring Boot Admin

``` java
  <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-starter-server</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-server-ui</artifactId>
   </dependency>
    <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
   </dependency>
```
### 启动类添加如下注解

``` java
@SpringBootApplication
@EnableAdminServer
public class SpringbootAdminApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringbootAdminApplication.class, args);
    }
}
```
### 添加身份验证和授权

``` java
@Configuration
public class SecuritySecureConfig extends WebSecurityConfigurerAdapter {



    private final String adminContextPath;

    public SecuritySecureConfig(AdminServerProperties adminServerProperties) {
        this.adminContextPath = adminServerProperties.getContextPath();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // @formatter:off
        SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
        successHandler.setTargetUrlParameter("redirectTo");
        successHandler.setDefaultTargetUrl(adminContextPath + "/");

        http.authorizeRequests()
                //授予对所有静态资产和登录页面的公共访问权限。
                .antMatchers(adminContextPath + "/assets/**").permitAll()
                .antMatchers(adminContextPath + "/login").permitAll()
                //必须对每个其他请求进行身份验证
                .anyRequest().authenticated()
                .and()
                //	配置登录和注销。
                .formLogin().loginPage(adminContextPath + "/login").successHandler(successHandler).and()
                .logout().logoutUrl(adminContextPath + "/logout").and()
                //启用HTTP-Basic支持。这是Spring Boot Admin Client注册所必需的
                .httpBasic().and()
                .csrf()
                //使用Cookie启用CSRF保护
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .ignoringAntMatchers(
                        adminContextPath + "/instances", //禁用CRSF-Protection Spring Boot Admin Client用于注册的端点。
                        adminContextPath + "/actuator/**" //
                );
        // @formatter:on
    }
}

```

### application.properties配置文件

``` bash
server.port=8088
server.tomcat.uri-encoding=UTF-8
server.tomcat.max-threads=1000
server.tomcat.min-spare-threads=30
#账户密码
spring.security.user.name=gzpflm
spring.security.user.password=gzpflm
#项目访问名
spring.boot.admin.context-path=/szq-monitoring
#UI界面标题
spring.boot.admin.ui.title=szq-Monitpring

```
启动运行：http://localhost:8088/szq-monitoring/login 出现登录界面表示成功
![](http://wx1.sinaimg.cn/large/006b7Nxngy1g1g3byk2lqj30st0j4q3n.jpg)

### Spring Boot客户端配置监控
- 客户端需要配置账户密码 不然无法注册到springBoot Admin
- 每个要注册的应用程序都必须包含Spring Boot Admin Client 配置如下
``` java
   <dependency>
            <groupId>de.codecentric</groupId>
            <artifactId>spring-boot-admin-starter-client</artifactId>
  </dependency>
```
 **application.properties配置文件** 

``` bash
server.port=8081
spring.application.name=Spring Boot Client
spring.boot.admin.client.url=http://localhost:8088/szq-monitoring
management.endpoints.web.exposure.include=*
spring.boot.admin.client.username=gzpflm
spring.boot.admin.client.password=gzpflm
spring.boot.admin.client.enabled=true
#启用ip显示
spring.boot.admin.client.instance.prefer-ip=true
```
启动后：监控的服务端就会收到通知 刷新页面就可以看到监控的服务
![](http://wx1.sinaimg.cn/large/006b7Nxngy1g1g3cmpm60j31540lzdh4.jpg)

![](http://wx1.sinaimg.cn/large/006b7Nxngy1g1g3dc88adj31af0nowga.jpg)



项目地址：https://gitee.com/qinxuewu/SpringBoot--Admin-demo


### Spring Boot Admin Client配置选项

``` bash
spring.boot.admin.client.enabled    #启用S​​pring Boot Admin Client,默认值true
spring.boot.admin.client.url  #逗号分隔Spring Boot Admin服务器的有序URL列表以进行注册
spring.boot.admin.client.api-path #管理服务器上的注册端点的Http路径 默认值"instances"
#SBA Server api受HTTP基本身份验证保护时的用户名和密码。
spring.boot.admin.client.username 
spring.boot.admin.client.password
spring.boot.admin.client.period #重复注册的间隔（以ms为单位）默认自10,000
spring.boot.admin.client.connect-timeout  #连接超时进行注册（以ms为单位 #默认5,000
```
### 官方配置
http://codecentric.github.io/spring-boot-admin/current/#register-clients-via-spring-boot-admin

![觉得本文不错的话，分享一下给小伙伴吧~](http://wx1.sinaimg.cn/large/006b7Nxngy1g1eu6ewhl9j30760763yz.jpg)