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
`* 双VIP互为主备`  
`* KeepAlived Http接口健康探测`   
`* LVS DR直接路由`    
`* 透传CIP 响应报文Ingress直接返回`    

## 3.访问流程
![](/img/2023-03-24-keepalived+lvs/lvs01.jpg)

<font size=2>1.当客户端用户发送请求http://www.xxx.com/，首先经过DNS解析到IP后经过网络到达keepalived服务器。 </font>
<p></p>
<font size=2>2.此时达到 keepalived 网卡的数据包包括：SIP(客户端地址)、DIP(keepalived-vip)、SMAC(cmac/keepalived连接路由的mac)、DMAC(vip对应的mac)。</font>
<p></p>
<font size=2>3.数据包达到网卡后，经过链路层到达 PREROUTING 链，进行查找路由，发现 DIP 是lvs的 VIP，这时就会发送至 INPUT 链中并且数据包的IP地址、MAC地址、Port都未经过修改。 </font>
<p></p>
<font size=2>4.数据包到到达INPUT链中，LVS会根据目的IP和Port确认是否为LVS定义的服务。（如果不是则直接进入用户空间）。   </font>
<p></p>
<font size=2>5.如果是定义过的VIP服务，会根据配置的服务信息及相关算法（默认rr）从 RealServer 中选择一个后端服务，被 ipvs 规则强行扭转到 POSTROUTING 链，此时 SMAC 为VIP所在网卡MAC，目标MAC为RS的MAC地址。   </font>
<p></p>
<font size=2>6.由于集群主机间都要接入交换机，而交换机只是识别MAC，所以数据包只在lvs上改了SMAC＋DMAC，其它层次并未改变。   </font>
<p></p>
<font size=2>7.当数据包达到RS内部时，看到是自己的MAC，且自己有VIP，那么开始构建响应报文，将响应报文通过本地配置路由规则（route add -host $VIP dev lo:0）交给lo接口传送给物理网卡向外发出。 </font>
<p></p>
<font size=2>8.此时的源 IP 地址为 VIP，目标 IP 为 CIP，源 MAC 地址为 RS1 的 RMAC，目的 MAC 地址为下一跳路由器的 MAC 地址（CMAC），最终数据包通过 RS 相连的路由器转发给客户端。   </font>



## 4.优劣势
#### 4.1 优势
`* 响应数据不经过lvs，性能高，可透传CIP。`   
`* 对数据包修改下，信息完整性好。`   
#### 4.2 劣势
`* LVS与RS通过二层协议进行转发，根据MAC地址来进行匹配寻址，需要在同一物理网络。`  
`* RS上必须配置lo与其它arp协议相关内核参数。`  
`* 不支持端口映射。`  
## 5.部署流程
#### 5.1 keepalived01配置
```shell
#baoding-lvs-10-255-23-42
[root@baoding-lvs-10-255-23-42 ~]# cat /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state MASTER
    interface bond0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 11112222
    }
    virtual_ipaddress {
        10.255.23.73
    }
}
vrrp_instance VI_2 {
    state BACKUP
    interface bond0
    virtual_router_id 52
 
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 11112222
    }
    virtual_ipaddress {
        10.255.23.74
    }
}
 
[root@baoding-lvs-10-255-23-42 ~]# systemctl start keepalived.service
[root@baoding-lvs-10-255-23-42 ~]# systemctl enable keepalived.service
```
#### 5.2 keepalived02配置
```shell
#baoding-lvs-10-255-23-44
[root@baoding-lvs-10-255-23-44 ~]# cat /etc/keepalived/keepalived.conf
vrrp_instance VI_1 {
    state BACKUP
    interface bond0
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 11112222
    }
    virtual_ipaddress {
        10.255.23.73
    }
}
vrrp_instance VI_2 {
    state MASTER
    interface bond0
    virtual_router_id 52
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 11112222
    }
    virtual_ipaddress {
        10.255.23.74
    }
}
[root@baoding-lvs-10-255-23-44 ~]# systemctl start keepalived.service
[root@baoding-lvs-10-255-23-44 ~]# systemctl enable keepalived.service
```
#### 5.3 keepalived-tcp健康探测
```shell
#tcp健康探测
virtual_server 10.0.0.100 80 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    persistence_timeout 120
    protocol TCP
    real_server 10.0.0.13 80 {
        weight 1
        TCP_CHECK {   
            connect_timeout 5
            nb_get_retry 3   
            delay_before_retry 3
        }
    }
    real_server 10.0.0.14 80 {
        weight 1
        TCP_CHECK {       
            connect_timeout 5
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
```
#### 5.4 keepalived-http健康探测
```shell
#http健康检查
virtual_server 10.255.23.73 8765 {
    delay_loop 10
    lb_algo rr
    lb_kind DR
    persistence_timeout 120
    protocol TCP
    real_server 10.255.23.4 8765 {
        weight 1
        HTTP_GET {
            url {
                path /index.html
                status_code 200
            }
            connect_timeout 5
            nb_get_retry 3
            delay_before_retry 3
        }
    }
    real_server 10.255.23.10 8765 {
        weight 1
        HTTP_GET {
            url {
                path /index.html
                status_code 200
            }
            connect_timeout 5
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
 
virtual_server 10.255.23.74 8765 {
    delay_loop 10
    lb_algo rr
    lb_kind DR
    persistence_timeout 120
    protocol TCP
    real_server 10.255.23.4 8765 {
        weight 1
        HTTP_GET {
            url {
                path /index.html
                status_code 200
            }
            connect_timeout 5
            nb_get_retry 3
            delay_before_retry 3
        }
    }
    real_server 10.255.23.10 8765 {
        weight 1
        HTTP_GET {
            url {
                path /index.html
                status_code 200
            }
            connect_timeout 5
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
virtual_server 10.255.23.73 8765 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    persistence_timeout 120
    protocol TCP
    real_server 10.255.23.4 8765 {
        weight 1
        HTTP_GET {
            url {
                path /index.html
                status_code 200
            }
            connect_timeout 5
            nb_get_retry 3
            delay_before_retry 3
        }
    }
    real_server 10.255.23.10 8765 {
        weight 1
        HTTP_GET {
            url {
                path /index.html
                status_code 200
            }
            connect_timeout 5
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
 
virtual_server 10.255.23.74 8765 {
    delay_loop 6
    lb_algo rr
    lb_kind DR
    persistence_timeout 120
    protocol TCP
    real_server 10.255.23.4 8765 {
        weight 1
        HTTP_GET {
            url {
                path /index.html
                status_code 200
            }
            connect_timeout 5
            nb_get_retry 3
            delay_before_retry 3
        }
    }
    real_server 10.255.23.10 8765 {
        weight 1
        HTTP_GET {
            url {
                path /index.html
                status_code 200
            }
            connect_timeout 5
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
```
#### 5.5 开启IP路由转发功能
```shell
# 两台keepalived均执行 开启内核IP路由转发功能
[root@baoding-lvs-10-255-23-42 ~]# yum install -y ipvsadm
[root@baoding-lvs-10-255-23-42 ~]# echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
[root@baoding-lvs-10-255-23-42 ~]# sysctl -p
net.ipv4.ip_forward = 1
```
#### 5.6 创建virtual-service和real-server
```shell
#两台vs均执行
[root@baoding-lvs-10-255-23-42 ~]# ipvsadm -A -t 10.255.23.73:80 -s rr
[root@baoding-lvs-10-255-23-42 ~]# ipvsadm -A -t 10.255.23.74:80 -s rr
[root@baoding-lvs-10-255-23-42 ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.255.23.73:80 rr
TCP  10.255.23.74:80 rr
[root@baoding-lvs-10-255-23-42 ~]# ipvsadm -a -t 10.255.23.73:80 -r 10.255.23.4:80 -g
[root@baoding-lvs-10-255-23-42 ~]# ipvsadm -a -t 10.255.23.73:80 -r 10.255.23.9:80 -g
[root@baoding-lvs-10-255-23-42 ~]# ipvsadm -a -t 10.255.23.74:80 -r 10.255.23.9:80 -g
[root@baoding-lvs-10-255-23-42 ~]# ipvsadm -a -t 10.255.23.74:80 -r 10.255.23.4:80 -g
[root@baoding-lvs-10-255-23-42 ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.255.23.73:80 rr
  -> 10.255.23.4:80               Route   1      0          0        
  -> 10.255.23.9:80               Route   1      0          0        
TCP  10.255.23.74:80 rr
  -> 10.255.23.4:80               Route   1      0          0        
  -> 10.255.23.9:80               Route   1      0          0
```
#### 5.7 rs配置网卡和路由
```shell
#两台rs均执行
 
#配置网卡
[root@master3 ~]# cp /etc/sysconfig/network-scripts/ifcfg-lo /etc/sysconfig/network-scripts/ifcfg-lo:1
[root@master3 ~]# cp /etc/sysconfig/network-scripts/ifcfg-lo /etc/sysconfig/network-scripts/ifcfg-lo:2
[root@master3 ~]# cat /etc/sysconfig/network-scripts/ifcfg-lo:1
DEVICE=lo:1
IPADDR=10.255.23.73
NETMASK=255.255.255.255
NETWORK=127.0.0.0
# If you're having problems with gated making 127.0.0.0/8 a martian,
# you can change this to something else (255.255.255.255, for example)
BROADCAST=127.255.255.255
ONBOOT=yes
NAME=loopback
[root@master3 ~]# cat /etc/sysconfig/network-scripts/ifcfg-lo:2
DEVICE=lo:2
IPADDR=10.255.23.74
NETMASK=255.255.255.255
NETWORK=127.0.0.0
# If you're having problems with gated making 127.0.0.0/8 a martian,
# you can change this to something else (255.255.255.255, for example)
BROADCAST=127.255.255.255
ONBOOT=yes
NAME=loopback
[root@master3 ~]# ifup lo:1
[root@master3 ~]# ifup lo:2
[root@master3 ~]# ip addr|head
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet 10.255.23.73/32 brd 127.255.255.255 scope global lo:1
       valid_lft forever preferred_lft forever
    inet 10.255.23.74/32 brd 127.255.255.255 scope global lo:2
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
 
#配置内核参数 arp_ignore(响应arp作用域/级别) arp_announce(发送arp源信息) 
[root@localhost network-scripts]# cat >> /etc/sysctl.conf << EOF
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.default.arp_ignore = 1
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce = 2
EOF
[root@localhost network-scripts]# sysctl -p
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.default.arp_ignore = 1
net.ipv4.conf.lo.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce = 2
 
#配置路由
[root@master3 ~]# route add -host 10.255.23.73 dev lo:1
[root@master3 ~]# route add -host 10.255.23.74 dev lo:2
[root@master3 ~]# ip route
default via 10.255.23.254 dev bond0
10.255.23.0/24 dev bond0 proto kernel scope link src 10.255.23.4
10.255.23.73 dev lo scope link src 10.255.23.73
10.255.23.74 dev lo scope link src 10.255.23.74
169.254.0.0/16 dev bond0 scope link metric 1006
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
179.20.0.0/24 via 179.20.0.0 dev flannel.1 onlink
179.20.1.0/24 dev cni0 proto kernel scope link src 179.20.1.1
179.20.2.0/24 via 179.20.2.0 dev flannel.1 onlink
179.20.3.0/24 via 179.20.3.0 dev flannel.1 onlink
179.20.4.0/24 via 179.20.4.0 dev flannel.1 onlink
179.20.5.0/24 via 179.20.5.0 dev flannel.1 onlink
179.20.6.0/24 via 179.20.6.0 dev flannel.1 onlink
179.20.7.0/24 via 179.20.7.0 dev flannel.1 onlink
179.20.8.0/24 via 179.20.8.0 dev flannel.1 onlink
179.20.9.0/24 via 179.20.9.0 dev flannel.1 onlink
179.20.10.0/24 via 179.20.10.0 dev flannel.1 onlink
[root@master3 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.255.23.254   0.0.0.0         UG    0      0        0 bond0
10.255.23.0     0.0.0.0         255.255.255.0   U     0      0        0 bond0
10.255.23.73    0.0.0.0         255.255.255.255 UH    0      0        0 lo
10.255.23.74    0.0.0.0         255.255.255.255 UH    0      0        0 lo
169.254.0.0     0.0.0.0         255.255.0.0     U     1006   0        0 bond0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
179.20.0.0      179.20.0.0      255.255.255.0   UG    0      0        0 flannel.1
179.20.1.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
179.20.2.0      179.20.2.0      255.255.255.0   UG    0      0        0 flannel.1
179.20.3.0      179.20.3.0      255.255.255.0   UG    0      0        0 flannel.1
179.20.4.0      179.20.4.0      255.255.255.0   UG    0      0        0 flannel.1
179.20.5.0      179.20.5.0      255.255.255.0   UG    0      0        0 flannel.1
179.20.6.0      179.20.6.0      255.255.255.0   UG    0      0        0 flannel.1
179.20.7.0      179.20.7.0      255.255.255.0   UG    0      0        0 flannel.1
179.20.8.0      179.20.8.0      255.255.255.0   UG    0      0        0 flannel.1
179.20.9.0      179.20.9.0      255.255.255.0   UG    0      0        0 flannel.1
179.20.10.0     179.20.10.0     255.255.255.0   UG    0      0        0 flannel.1
```
## 6.测试过程
#### 6.1 keepalived宕机测试
```shell
#keepalived宕机测试
[root@baoding-lvs-10-255-23-42 ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
8: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 68:05:ca:ea:91:b0 brd ff:ff:ff:ff:ff:ff
    inet 10.255.23.42/24 brd 10.255.23.255 scope global noprefixroute bond0
       valid_lft forever preferred_lft forever
    inet 10.255.23.73/32 scope global bond0
       valid_lft forever preferred_lft forever
    inet6 fe80::6664:5f65:fa5c:5269/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
 
[root@baoding-lvs-10-255-23-44 ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
8: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 68:05:ca:ea:96:ec brd ff:ff:ff:ff:ff:ff
    inet 10.255.23.44/24 brd 10.255.23.255 scope global noprefixroute bond0
       valid_lft forever preferred_lft forever
    inet 10.255.23.74/32 scope global bond0
       valid_lft forever preferred_lft forever
    inet6 fe80::2eed:a810:598d:89bf/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
 
#宕机10.255.23.73
[root@baoding-lvs-10-255-23-42 ~]# systemctl stop keepalived.service
[root@baoding-lvs-10-255-23-42 ~]#
[root@baoding-lvs-10-255-23-42 ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
8: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 68:05:ca:ea:91:b0 brd ff:ff:ff:ff:ff:ff
    inet 10.255.23.42/24 brd 10.255.23.255 scope global noprefixroute bond0
       valid_lft forever preferred_lft forever
    inet6 fe80::6664:5f65:fa5c:5269/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
[root@baoding-lvs-10-255-23-44 ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
8: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 68:05:ca:ea:96:ec brd ff:ff:ff:ff:ff:ff
    inet 10.255.23.44/24 brd 10.255.23.255 scope global noprefixroute bond0
       valid_lft forever preferred_lft forever
    inet 10.255.23.74/32 scope global bond0
       valid_lft forever preferred_lft forever
    inet 10.255.23.73/32 scope global bond0
       valid_lft forever preferred_lft forever
    inet6 fe80::2eed:a810:598d:89bf/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
#vip故障已转移，测试服务连通性
[root@master3 ~]# dig shanyi-test01.preview.paas.gwm.cn
 
; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.13 <<>> shanyi-test01.preview.paas.gwm.cn
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5497
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
 
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4000
;; QUESTION SECTION:
;shanyi-test01.preview.paas.gwm.cn. IN  A
 
;; ANSWER SECTION:
shanyi-test01.preview.paas.gwm.cn. 60 IN A  10.255.23.73
shanyi-test01.preview.paas.gwm.cn. 60 IN A  10.255.23.74
 
;; Query time: 4 msec
;; SERVER: 10.255.18.3#53(10.255.18.3)
;; WHEN: Thu Mar 23 15:07:02 CST 2023
;; MSG SIZE  rcvd: 94
[root@bd-ywzt-paas-cs01 ~]# curl http://shanyi-test01.preview.paas.gwm.cn/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
 
<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
 
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
#### 6.2 real-server环境模拟
```shell
#模拟网关故障问题
[root@baoding-lvs-10-255-23-42 ~]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.255.23.73:8765 rr
  -> 10.255.23.4:8765             Route   1      0          0        
  -> 10.255.23.10:8765            Route   1      0          0        
TCP  10.255.23.74:8765 rr
  -> 10.255.23.4:8765             Route   1      0          0        
  -> 10.255.23.10:8765            Route   1      0          0   
 
[root@master2 ~]# netstat -lntp|egrep 8765
tcp6       0      0 :::8765                 :::*                    LISTEN      28037/docker-proxy 
[root@master3 ~]# netstat -lntp|egrep 8765
tcp6       0      0 :::8765                 :::*                    LISTEN      10135/docker-proxy 
 
[root@bd-ywzt-paas-cs01 ~]# curl http://shanyi-test01.preview.paas.gwm.cn:8765/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
 
<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
 
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
 
#10.255.23.4日志
10.255.30.245 - - [23/Mar/2023:07:37:29 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
10.255.30.245 - - [23/Mar/2023:07:37:34 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
 
#10.255.23.10日志
10.255.30.245 - - [23/Mar/2023:07:39:28 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
10.255.30.245 - - [23/Mar/2023:07:39:30 +0000] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
```
#### 6.3 real-server宕机测试
```shell
[root@baoding-lvs-10-255-23-42 conf]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.255.23.73:8765 rr persistent 120
  -> 10.255.23.4:8765             Route   1      0          0        
  -> 10.255.23.10:8765            Route   1      0          0        
TCP  10.255.23.74:8765 rr persistent 120
  -> 10.255.23.4:8765             Route   1      0          0        
  -> 10.255.23.10:8765            Route   1      0          0  
 
#停掉一台rs后，ipvs规则自动剔除非健康rs路由
[root@baoding-lvs-10-255-23-42 conf]# ipvsadm -Ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.255.23.73:8765 rr persistent 120
  -> 10.255.23.4:8765             Route   1      0          0        
TCP  10.255.23.74:8765 rr persistent 120
  -> 10.255.23.4:8765             Route   1      0          0
 
#访问服务正常
```