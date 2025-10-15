---
abbrlink: ''
categories: []
date: '2025-10-15T22:04:58.550+08:00'
tags: []
title: 腾讯云轻量应用服务器无法访问外部ipv6链接修复
updated: '2025-10-15T22:04:58.550+08:00'
---
## 一、问题描述

1. 已在腾讯云控制台开启公网 IPv6 功能，分配地址为【服务器 IPv6 地址】。
2. 外部设备可通过 IPv6 正常访问服务器，本地 ping 测试结果如下：
   shell

   ```shell
   root@fnOS:~# sudo ping 【服务器IPv6地址】                        
   PING 【服务器IPv6地址】(【服务器IPv6地址】) 56 data bytes  
   64 bytes from 【服务器IPv6地址】: icmp_seq=1 ttl=64 time=0.250 ms  
   64 bytes from 【服务器IPv6地址】: icmp_seq=2 ttl=64 time=0.300 ms  
   64 bytes from 【服务器IPv6地址】: icmp_seq=3 ttl=64 time=0.288 ms  
   64 bytes from 【服务器IPv6地址】: icmp_seq=4 ttl=64 time=0.305 ms  
   64 bytes from 【服务器IPv6地址】: icmp_seq=5 ttl=64 time=0.287 ms  
   ^C                                                                                 
   --- 【服务器IPv6地址】 ping statistics ---                       
   5 packets transmitted, 5 received, 0% packet loss, time 4085ms                     
   rtt min/avg/max/mdev = 0.250/0.286/0.305/0.019 ms
   ```
3. 服务器内部无法访问外部 IPv6 资源，curl 测试失败：
   shell

   ```shell
   $ curl -v http://【此处省略】.cn   
   *   Trying 【用于测试的外部ipv6地址】...
   * Immediate connect fail for 2408:8226:811:bfd0:10cc:a1ad:c4a7:2f8a: Network is unreachable
   * Closing connection 0
   curl: (7) Couldn't connect to server
   ```

## 二、解决方案

服务器 IPv6 出网失败的核心原因是**缺少正确的 IPv6 地址配置或默认路由**，需按以下两步排查修复：

### 1. 检查并配置本地 IPv6 地址

* 执行命令查看当前 IPv6 地址分配情况：
  shell

  ```shell
  ip addr | grep inet6
  ```
* 若未找到【服务器 IPv6 地址】，执行以下命令手动添加（`eth0`为网卡名，需根据实际调整）：
  shell

  ```shell
  sudo ip -6 addr add 【服务器IPv6地址】/64 dev eth0
  ```

### 2. 检查并配置 IPv6 默认路由

* 执行命令查看当前 IPv6 路由表：
  shell

  ```shell
  ip -6 route
  ```
* 若输出中无 `default`开头的路由，执行以下命令添加默认网关（网关地址需从腾讯云控制台获取）：
  shell

  ```shell
  sudo ip -6 route add default via 【IPv6网关地址】 dev eth0
  ```

## 三、具体修复过程

### 步骤 1：检查并补充 IPv6 地址

1. 执行 `ip addr | grep inet6`查看地址，发现已分配部分地址，但缺少【服务器 IPv6 地址】：
   shell

   ```shell
   # root @ VM-24-15-ubuntu in ~ [20:46:12] C:2
   $ ip addr | grep inet6
       inet6 ::1/128 scope host 
       inet6 【省略的IPv6地址】 scope global dynamic noprefixroute 
       inet6 【省略的IPv6地址】/64 scope link 
       inet6 【省略的IPv6地址】/64 scope link 
       # （其余省略地址均用【省略的IPv6地址】占位）
   ```
2. 手动添加【服务器 IPv6 地址】：
   shell

   ```shell
   # root @ VM-24-15-ubuntu in ~ [20:48:36] 
   $ sudo ip -6 addr add 【服务器IPv6地址】/64 dev eth0
   ```
3. 再次检查，确认地址已添加：
   shell

   ```shell
   # root @ VM-24-15-ubuntu in ~ [20:49:37] C:7
   $ ip addr | grep inet6                                          
       inet6 ::1/128 scope host 
       inet6 【服务器IPv6地址】/64 scope global  # 已成功添加
       inet6 【省略的IPv6地址】/128 scope global dynamic noprefixroute 
       # （其余省略地址均用【省略的IPv6地址】占位）
   ```

### 步骤 2：测试并修复 IPv6 路由

1. 尝试访问外部 IPv6 资源，仍失败（路由未配置）：
   shell

   ```shell
   # root @ VM-24-15-ubuntu in ~ [20:49:29] 
   $ curl -v http://【此处省略】.cn   
   *   Trying 【用于测试的外部ipv6地址】...
   * Immediate connect fail for 2408:8226:811:bfd0:10cc:a1ad:c4a7:2f8a: Network is unreachable
   * Closing connection 0
   curl: (7) Couldn't connect to server
   ```
2. 查看 IPv6 路由表，确认无默认路由：
   shell

   ```shell
   # root @ VM-24-15-ubuntu in ~ [20:50:16] C:130
   $ ip -6 route
   ::1 dev lo proto kernel metric 256 pref medium
   【省略的IPv6地址】/64 dev eth0 proto kernel metric 256 pref medium  # 无default路由
   fd76:3600:140a:9406::/64 dev eth0 proto ra metric 100 pref medium
   fe80::/64 dev eth0 proto kernel metric 256 pref medium
   # （其余fe80::/64相关路由省略）
   ```
3. 添加默认路由：
   shell

   ```shell
   # root @ VM-24-15-ubuntu in ~ [20:51:21] 
   $ sudo ip -6 route add default via 【IPv6网关地址】 dev eth0
   ```

### 步骤 3：验证修复结果

添加路由后再次测试，可正常访问外部 IPv6 资源，修复成功：

shell

```shell
# root @ VM-24-15-ubuntu in ~ [20:52:23] 
$ curl -v http://【此处省略】.cn
*   Trying 【用于测试的外部ipv6地址】...
* Connected to 【此处省略】.cn (【用于测试的外部ipv6地址】) port 80 (#0)
> GET / HTTP/1.1
> Host:【此处省略】.cn
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 401 Unauthorized
< Server: nginx
< Date: Sun, 12 Oct 2025 12:52:34 GMT
< Content-Type: text/plain; charset=utf-8
< Content-Length: 15
< Connection: keep-alive
< Www-Authenticate: Basic realm="Restricted"
< X-Content-Type-Options: nosniff
< 
Not Authorized
* Connection #0 to host fnwebdav.oilu.cn left intact
```

## 四、配置持久化（避免重启失效）

手动配置的 IPv6 地址和路由会在服务器重启后丢失，需通过 netplan 配置文件实现永久生效。

### 步骤 1：编辑 netplan 配置文件

1. 进入 netplan 配置目录，找到对应配置文件（通常为 `50-cloud-init.yaml`或 `01-netcfg.yaml`）：
   shell

   ```shell
   cd /etc/netplan/
   ls  # 查看目录下的配置文件
   sudo vim 50-cloud-init.yaml  # 编辑配置文件（文件名以实际为准）
   ```
2. 替换原配置内容为以下内容（隐藏敏感信息，实际使用时替换占位符）：
   yaml

   ```yaml
   network:
       version: 2
       ethernets:
           eth0:
               dhcp4: true  # 保留IPv4动态获取
               dhcp6: false  # 关闭IPv6动态获取（手动配置更稳定）
               addresses:
                   - 【服务器IPv6地址】/64  # 手动指定服务器IPv6地址
               routes:
                   - to: default  # 添加默认路由
                     via: 【IPv6网关地址】  # 腾讯云IPv6网关地址
               nameservers:
                   addresses:
                       - 【IPv6网关地址】  # 网关作为主DNS
                       - 2001:4860:4860::8888  # 谷歌公共IPv6 DNS（备用）
               match:
                   macaddress: 【网卡MAC地址】  # 绑定网卡（从`ip addr`查看）
               set-name: eth0  # 网卡名称
   ```

### 步骤 2：应用 netplan 配置

执行以下命令使配置生效，无需重启服务器：

shell

```shell
sudo netplan apply
```

### 步骤 3：验证持久化效果

重启服务器后，执行以下命令确认 IPv6 地址和路由是否保留：

shell

```shell
# 确认地址存在
ip addr | grep 【服务器IPv6地址】
# 确认默认路由存在
ip -6 route | grep default
```
