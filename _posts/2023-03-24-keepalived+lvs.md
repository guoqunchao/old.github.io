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

## 前言
本次通过双vip互为主备搭建keepalived、lvs、ingress高可用架构。

## 1.资源清单
| vip | lvs | ingress |
| --- | --- | --- |
| 10.255.23.73 | 10.255.23.42 | 10.255.23.4(预发布) |
| 10.255.23.74 | 10.255.23.44 | 10.255.23.9(预发布) |


## 2.架构拓扑
![](/img/2023-03-24-keepalived+lvs/lvs工作流程图2.jpg)
#### 2.1 双VIP互为主备
#### 2.2 KeepAlived Http接口健康探测
#### 2.3 LVS DR直接路由
#### 2.4 透传CIP 响应报文Ingress直接返回

## 3.访问流程
![](/img/2023-03-24-keepalived+lvs/lvs01.jpg)
1.当客户端用户发送请求http://www.xxx.com/，首先经过DNS解析到IP后经过网络到达keepalived服务器。 
2.此时达到 keepalived 网卡的数据包包括：SIP(客户端地址)、DIP(keepalived-vip)、SMAC(cmac/keepalived连接路由的mac)、DMAC(vip对应的mac)。 
3.数据包达到网卡后，经过链路层到达 PREROUTING 链，进行查找路由，发现 DIP 是lvs的 VIP，这时就会发送至 INPUT 链中并且数据包的IP地址、MAC地址、Port都未经过修改。 
4.数据包到到达INPUT链中，LVS会根据目的IP和Port确认是否为LVS定义的服务。（如果不是则直接进入用户空间）。 
5.如果是定义过的VIP服务，会根据配置的服务信息及相关算法（默认rr）从 RealServer 中选择一个后端服务，被 ipvs 规则强行扭转到 POSTROUTING 链，此时 SMAC 为VIP所在网卡MAC，目标MAC为RS的MAC地址。 
6.由于集群主机间都要接入交换机，而交换机只是识别MAC，所以数据包只在lvs上改了SMAC＋DMAC，其它层次并未改变。 
7.当数据包达到RS内部时，看到是自己的MAC，且自己有VIP，那么开始构建响应报文，将响应报文通过本地配置路由规则（route add -host $VIP dev lo:0）交给lo接口传送给物理网卡向外发出。 
8.此时的源 IP 地址为 VIP，目标 IP 为 CIP，源 MAC 地址为 RS1 的 RMAC，目的 MAC 地址为下一跳路由器的 MAC 地址（CMAC），最终数据包通过 RS 相连的路由器转发给客户端。 

## 4.优劣势
#### 4.1 优势
`响应数据不经过lvs，性能高，可透传CIP。` 
`对数据包修改下，信息完整性好。` 
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