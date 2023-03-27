---
layout:     post
title:      KeepAlived+LVS
subtitle:   keepalived+lvs+ingress配置详解
date:       2023-03-24
author:     shanyi
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - KeepAlived
    - LVS
    - Ingress
    - ipvs
    - iptables
---

## 前言
本次通过双vip互为主备搭建keepalived、lvs、ingress高可用架构。

## 1.资源清单
| vip | lvs | ingress |
| --- | --- | --- |
| 10.255.23.73 | 10.255.23.42 | 10.255.23.4(预发布) |
| 10.255.23.74 | 10.255.23.44 | 10.255.23.9(预发布) |

## 2.架构拓扑
#### 2.1 双VIP互为主备
#### 2.2 KeepAlived Http接口健康探测
#### 2.3 LVS DR直接路由
#### 2.4 透传CIP 响应报文Ingress直接返回

## 3.访问流程
img/lvs01.jpg
