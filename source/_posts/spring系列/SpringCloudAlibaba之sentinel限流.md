---
title: spring cloud alibaba系列(二)Sentinel应用的限流管理
date: 2019-01-28 14:55:45
tags: 
- springcloud 
- alibaba
categories: springcloud

---

## 限流组件Sentinel
- Sentinel是把流量作为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性。
- 默认支持 Servlet、Feign、RestTemplate、Dubbo 和 RocketMQ 限流降级功能的接入，可以在运行时通过控制台实时修改限流降级规则，还支持查看限流降级 Metrics 监控。
- 自带控台动态修改限流策略。但是每次服务重启后就丢失了。所以它也支持ReadableDataSource 目前支持file, nacos, zk, apollo 这4种类型
## 接入Sentinel
创建项目cloud-sentinel
* 1 引入 Sentinel starter
``` java
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```
* 2 application.properties配置如下
``` bash
server.port=18084
spring.application.name=service-sentinel

#Sentinel 控制台地址
spring.cloud.sentinel.transport.dashboard=localhost:8080
#取消Sentinel控制台懒加载
spring.cloud.sentinel.eager=true
```
## 接入限流埋点
Sentinel 默认为所有的 HTTP 服务提供了限流埋点。引入依赖后自动完成所有埋点。只需要在控制配置限流规则即可
* 注解埋点
  如果需要对某个特定的方法进行限流或降级，可以通过 @SentinelResource 注解来完成限流的埋点

``` java
@SentinelResource("resource")
@RequestMapping("/sentinel/hello")
public Map<String,Object> hello(){
        Map<String,Object> map=new HashMap<>(2);
        map.put("appName",appName);
        map.put("method","hello");
        return map;
}
```
## 部署Sentinel控制台
### 安装
[Sentinel下载](http://edas-public.oss-cn-hangzhou.aliyuncs.com/install_package/demo/sentinel-dashboard.jar)
### 启动控制台
执行 Java 命令 `java -jar sentinel-dashboard.jar` 默认的监听端口为 `8080`
### 访问
打开http://localhost:8080 即可看到控制台界面
![输入图片说明](https://images.gitee.com/uploads/images/2019/0128/142828_12667ffe_1478371.png)
说明cloud-sentinel已经成功和Sentinel完成率通讯

## 配置限流规则
如果控制台没有找到自己的应用，可以先调用一下进行了 Sentinel 埋点的 URL 或方法或着禁用Sentinel 的赖加载`spring.cloud.sentinel.eager=true`
### 配置 URL 限流规则
控制器随便添加一个普通的http方法
``` Java
  /**
     * 通过控制台配置URL 限流
     * @return
     */
    @RequestMapping("/sentinel/test")
    public Map<String,Object> test(){
        Map<String,Object> map=new HashMap<>(2);
        map.put("appName",appName);
        map.put("method","test");
        return map;
    }

```
点击新增流控规则。为了方便测试阀值设为 1
![](http://wx1.sinaimg.cn/large/006b7Nxngy1g1g3lr3c52j31260h2myk.jpg)
浏览器重复请求 http://localhost:18084/sentinel/test 如果超过阀值就会出现如下界面
![](http://wx1.sinaimg.cn/large/006b7Nxngy1g1g3m94p13j30bn033mx4.jpg)

整个URL限流就完成了。但是返回的提示不够友好。

### 配置自定义限流规则(@SentinelResource埋点)
自定义限流规则就不是添加方法的访问路径。 配置的是@SentinelResource注解中value的值。比如` @SentinelResource("resource")`就是配置路径为resource
![](http://wx1.sinaimg.cn/large/006b7Nxngy1g1g3moc9tlj311q0f275h.jpg)

- 访问：http://localhost:18084/sentinel/hello
- 通过`@SentinelResource`注解埋点配置的限流规则如果没有自定义处理限流逻辑，当请求到达限流的阀值时就返回404页面
  ![](http://wx1.sinaimg.cn/large/006b7Nxngy1g1g3n57o7yj30ho07qaan.jpg)

## 自定义限流处理逻辑
@SentinelResource 注解包含以下属性：
- value: 资源名称，必需项（不能为空）
- entryType: 入口类型，可选项（默认为 EntryType.OUT）
- blockHandler:blockHandlerClass中对应的异常处理方法名。参数类型和返回值必须和原方法一致
- blockHandlerClass：自定义限流逻辑处理类
``` java

 //通过注解限流并自定义限流逻辑
 @SentinelResource(value = "resource2", blockHandler = "handleException", blockHandlerClass = {ExceptionUtil.class})
 @RequestMapping("/sentinel/test2")
    public Map<String,Object> test2() {
        Map<String,Object> map=new HashMap<>();
        map.put("method","test2");
        map.put("msg","自定义限流逻辑处理");
        return  map;
    }

public class ExceptionUtil {

    public static Map<String,Object> handleException(BlockException ex) {
        Map<String,Object> map=new HashMap<>();
        System.out.println("Oops: " + ex.getClass().getCanonicalName());
        map.put("Oops",ex.getClass().getCanonicalName());
        map.put("msg","通过@SentinelResource注解配置限流埋点并自定义处理限流后的逻辑");
        return  map;
    }
}
```
控制台新增resource2的限流规则并设置阀值为1。访问http://localhost:18084/sentinel/test2 请求到达阀值时机会返回自定义的异常消息

![](http://wx1.sinaimg.cn/large/006b7Nxngy1g1g3nm45lqj30ho07qaan.jpg)

基本的限流处理就完成了。 但是每次服务重启后 之前配置的限流规则就会被清空因为是内存态的规则对象.所以下面就要用到Sentinel一个特性ReadableDataSource 获取文件、数据库或者配置中心是限流规则
## 读取文件的实现限流规则
一条限流规则主要由下面几个因素组成：
* resource：资源名，即限流规则的作用对象
* count: 限流阈值
* grade: 限流阈值类型（QPS 或并发线程数）
* limitApp: 流控针对的调用来源，若为 default 则不区分调用来源
* strategy: 调用关系限流策略
* controlBehavior: 流量控制效果（直接拒绝、Warm Up、匀速排队）
  SpringCloud alibaba集成Sentinel后只需要在配置文件中进行相关配置，即可在 Spring 容器中自动注册 DataSource，这点很方便。配置文件添加如下配置

``` bash
#通过文件读取限流规则
spring.cloud.sentinel.datasource.ds1.file.file=classpath: flowrule.json
spring.cloud.sentinel.datasource.ds1.file.data-type=json
spring.cloud.sentinel.datasource.ds1.file.rule-type=flow
```
在resources新建一个文件 比如flowrule.json 添加限流规则

``` javascript
[
  {
    "resource": "resource",
    "controlBehavior": 0,
    "count": 1,
    "grade": 1,
    "limitApp": "default",
    "strategy": 0
  },
  {
    "resource": "resource3",
    "controlBehavior": 0,
    "count": 1,
    "grade": 1,
    "limitApp": "default",
    "strategy": 0
  }
]

```
 **重新启动项目。出现如下日志说明文件读取成功** 

``` Java
 [Sentinel Starter] DataSource ds1-sentinel-file-datasource start to loadConfig
 [Sentinel Starter] DataSource ds1-sentinel-file-datasource load 2 FlowRule
```

 **刷新Sentinel 控制台 限流规则就会自动添加进去** 
![](http://wx1.sinaimg.cn/large/006b7Nxngy1g1g3o3r1u3j318l063wey.jpg)

## Sentinel的基本配置

``` bash
spring.cloud.sentinel.enabled              #Sentinel自动化配置是否生效
spring.cloud.sentinel.eager               #取消Sentinel控制台懒加载
spring.cloud.sentinel.transport.dashboard   #Sentinel 控制台地址
spring.cloud.sentinel.transport.heartbeatIntervalMs        #应用与Sentinel控制台的心跳间隔时间
spring.cloud.sentinel.log.dir            #Sentinel 日志文件所在的目录
```

* 案例源码：<https://github.com/a870439570/alibaba-cloud>

![觉得本文不错的话，分享一下给小伙伴吧~](http://wx1.sinaimg.cn/large/006b7Nxngy1g1eu6ewhl9j30760763yz.jpg)