# 怎么配置网络
### 一、配置IP
1. 传统 /etc/sysconfig/network-scripts/ifcfg-etho
```
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=etho
DEVICE=etho
ONBOOT=yes
IPADDR=192.168.10.110
PREFIX=24
DNS1=114.114.114.114
IPV6_PRIVACY=no
```

2. ubuntu新版本 /etc/netplan/00-installer-config.yaml

```
# This is the network config written by 'subiquity'
network:
  ethernets:
    eth0:
      addresses:
      - 192.168.10.110/24
      gateway4: 192.168.10.1
      nameservers:
        addresses:
        - 114.114.114.114
        search:
        - df4j.com
  version: 2
```
### 二、配置路由
1. 临时配置路由信息
    + route
      ``` bash
      route add 192.168.10.0 netmask 255.255.255.0 gw 192.168.10.1
      route add 192.168.10.0/24 gw 192.168.10.1
      route add 192.168.10.0/24 dev etho
      ```
    + ip 
      ``` bash
      ip route add 192.168.10.0/24 via 192.168.10.1
      ip route add 192.168.10.0/24 via etho
      ```
2. 静态路由
配置文件 /etc/sysconfig/static-routes，由/etc/init.d/network 调用
```shell
    # Add non interface-specific static-routes.
    if [ -f /etc/sysconfig/static-routes ]; then
        if [ -x /sbin/route ]; then
            grep "^any" /etc/sysconfig/static-routes | while read ignore args ; do
                /sbin/route add -$args
            done
        else
            net_log $"Legacy static-route support not available: /sbin/route not found"
        fi
    fi
```
配置格式
```text
any net 192.168.1.0/24 gw 192.168.1.1
any net 192.168.2.0 netmask 255.255.255.0 gw 192.168.2.1
any host 10.19.190.11/32 gw 10.19.177.10
any host 10.19.190.12 gw 10.19.177.10
```
