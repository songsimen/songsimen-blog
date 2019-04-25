---
title: SpringBoot集成prometheus+Grafana监控
date: 2019-04-02 20:58:45
tags: 
- springboot
- prometheus
- Grafana
categories: springboot


---

 ## 概述
 * `Prometheus`是一个最初在SoundCloud上构建的开源系统监视和警报工具包 。

## 添加依赖
```java
    <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!--prometheus监控  https://prometheus.io/docs/introduction/overview/-->
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
            <version>1.1.3</version>
        </dependency>

```
## 配置文件
```bash
spring.application.name=SpringBootPrometheus
# 监控端点配置
# 自定义端点路径  将  /actuator/{id}为/manage/{id}
#management.endpoints.web.base-path=/manage
management.endpoints.web.exposure.include=*
management.metrics.tags.application=${spring.application.name}
```
## 启动类添加

```java
@SpringBootApplication
public class FreemarkerApplication {
    @Value("${spring.application.name}")
    private  String application;
    
    public static void main(String[] args) {
        SpringApplication.run(FreemarkerApplication.class, args);
    }
    @Bean
    MeterRegistryCustomizer<MeterRegistry> configurer() {
        return (registry) -> registry.config().commonTags("application", application);
    }
}
```
> 查看度量指标是否集成成功

浏览器访问：http://localhost:8081/actuator/prometheus

![启动成功](https://img-blog.csdnimg.cn/20190402132903893.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
## 安装Prometheus
* 下载地址：https://prometheus.io/download/  
* 选择时间序列数据库版本
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20190402133007321.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
> `Prometheus`会将所有采集到的样本数据以时间序列（time-series）的方式保存在内存数据库中，并且定时保存到硬盘上。




![解压](https://img-blog.csdnimg.cn/2019040213340280.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)



* Linux启动方式：`nohup /home/prometheus/prometheus2.8.1/prometheus &`

## 配置prometheus.yml

* SpringBoot官方配置：https://docs.spring.io/spring-boot/docs/2.1.3.RELEASE/reference/htmlsingle/
* prometheus官方文档：https://prometheus.io/docs/introduction/overview/
* Prometheus-配置解析: https://www.cnblogs.com/liujiliang/p/10080849.html

```bash
 # 全局配置
global:
  scrape_interval:     15s # 多久 收集 一次数据
  evaluation_interval: 15s # 多久评估一次 规则
  scrape_timeout:      10s   # 每次 收集数据的 超时时间

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# # 规则文件, 可以使用通配符
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']

# SpringBoot应用配置
  - job_name: 'SpringBootPrometheus'
    scrape_interval: 5s
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['127.0.0.1:8081']

```
## 启动Prometheus
> 浏览器访问：http://localhost:9090
> ![启动成功界面](https://img-blog.csdnimg.cn/20190402134739139.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
## 查看Prometheus监控的应用
![监控的应用](https://img-blog.csdnimg.cn/20190402135041685.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
* UP状态表示目前存活的实例
* > 查看具体的监控指标
  > ![](https://img-blog.csdnimg.cn/20190402135346932.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
## Grafana安装配置
* 下载地址:https://grafana.com/grafana/download
* 这里本机使用win系统：https://dl.grafana.com/oss/release/grafana-6.0.2.windows-amd64.zip

![下载解压](https://img-blog.csdnimg.cn/20190402190244641.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
> 启动   `grafana-server.exe`
> Linux启动方式：nohup /home/prometheus/prometheus2.8.1/prometheus &

浏览器访问：http://127.0.0.1:3000/login

![登录界面](https://img-blog.csdnimg.cn/20190402190347714.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)

**默认用户和密码均为`admin`**
### 添加数据源
在Data Sources选项中添加数据源
![搜索Prometheus数据源](https://img-blog.csdnimg.cn/201904021916327.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
* 设置`数据源的名称`（唯一的，可添加多个数据源）和`Prometheus`的访问地址，如果`Prometheus`有设置账号密码才可以访问，则需要在Auth模块勾选`Basuc Auth` 设置账号密码

![设置](https://img-blog.csdnimg.cn/20190402201837863.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)

### 导入仪表盘模板
* 模板地址：https://grafana.com/dashboards
* 在搜索框中搜索`Spring Boot`会检索出相关的模板，选择一个自己喜欢
  ![搜索](https://img-blog.csdnimg.cn/20190402203107140.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
  这里我选择我比较喜欢第三个和第五个。模板ID分别是`4701`，`6756`
  ![第三个](https://img-blog.csdnimg.cn/20190402203322934.png)
  ![倒数第二个](https://img-blog.csdnimg.cn/2019040220365069.png)
* 红框标注的部分就是项目中需要配置代码, 复制模板ID
  ![模板ID4701](https://img-blog.csdnimg.cn/20190402203353935.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
* 开始导入,输入模板ID 点击`Load`

![](https://img-blog.csdnimg.cn/20190402204059907.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
![选择导入](https://img-blog.csdnimg.cn/20190402203841680.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
* 设置
  ![](https://img-blog.csdnimg.cn/20190402204013516.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
* 添加完成
  ![添加完成](https://img-blog.csdnimg.cn/20190402204848531.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTAzOTEzNDI=,size_16,color_FFFFFF,t_70)
* Grafana还支持很多数据源的监控， 后续在慢慢研究

![觉得本文不错的话，分享一下给小伙伴吧~](http://wx1.sinaimg.cn/large/006b7Nxngy1g1eu6ewhl9j30760763yz.jpg)