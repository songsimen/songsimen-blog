---
title: SpringBoot版的JVM检控工具
date: 2019-01-26 17:46:51
tags: springboot
categories: springboot
# 如果top值为true，则会是首页推荐文章
top: true
---

## 项目介绍

- 基于SpringBoot2.0 实现的jvm远程监工图形化工具，可以同时监控多个web应用
- 该项目是借鉴另个一开源项目 （ JavaMonitor） https://gitee.com/zyzpp/JavaMonitor 演变而来，剔除了一些功能，增加了可远程监控模块，只需要在需要监控的项目集成监控的jar包 并设置可访问的IP（默认为空 则不拦截IP访问） 就可以实现远程监控,和用户管理模块,动态定时任务
支付windows服务器和Linux服务监控,Mac还未测试 应该也支持 

## 目录说明

1. boot-actuator  需要监控的项目demo
1. actuator-service  监控端点jar包 需要引入到需要监控的项目中（已打包好上传）
1. boot-monitor    监监控图形化工程
1. Sql文件  /boot-monitor/src/main/resources/db/actuator.sql

## [部署文档](https://a870439570.github.io/work-doc/actuator/)    

- https://a870439570.github.io/work-doc/actuator/

## 源码地址

https://github.com/a870439570/boot-actuator

## 部分效果图如下

### 监控列表主页  增加应用，删除应用

![](http://wx1.sinaimg.cn/large/006b7Nxngy1g1g3j279phj31ef0omjy6.jpg)

### 监控详情
![](http://wx1.sinaimg.cn/large/006b7Nxngy1g1g3jt5wz1j31dh0ohafa.jpg)



![觉得本文不错的话，分享一下给小伙伴吧~](http://wx1.sinaimg.cn/large/006b7Nxngy1g1eu6ewhl9j30760763yz.jpg)