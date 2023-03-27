---
layout:     post
title:      KeepAlived+LVS
subtitle:   keepalived+lvs+ingress配置详解
date:       2023-03-24
author:     shanyi
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - keepalived
    - lvs
    - ingress
    - ipvs
    - iptables
---

#### 前言
本次通过双vip互为主备搭建keepalived、lvs、ingress高可用架构。

#### 1.资源清单
| vip | lvs | ingress |  
| --- | --- | --- |    
| 10.255.23.73 | 10.255.23.42 | 10.255.23.4(预发布) |  
| 10.255.23.74 | 10.255.23.44 | 10.255.23.9(预发布) |  


#### 2.架构拓扑
![](/img/2023-03-24-keepalived+lvs/lvs工作流程图2.jpg)
##### 2.1 双VIP互为主备
##### 2.2 KeepAlived Http接口健康探测
##### 2.3 LVS DR直接路由
##### 2.4 透传CIP 响应报文Ingress直接返回

## 3.访问流程
![](/img/2023-03-24-keepalived+lvs/lvs01.jpg)

## 4.优劣势
#### 4.1 优势
#### 4.2 劣势
## 5.部署流程
#### 5.1 keepalived01配置
#### 5.2 keepalived02配置
#### 5.3 keepalived-tcp健康探测
#### 5.4 keepalived-http健康探测
#### 5.5 开启IP路由转发功能
#### 5.6 创建virtual-service和real-server
#### 5.7 rs配置网卡和路由
## 6.测试过程
#### 6.1 keepalived宕机测试
#### 6.2 real-server环境模拟
#### 6.3 real-server宕机测试