---
title: Dnsmasq 介绍与使用
date: 2017-05-23 14:06:00
categories: 知识整理/总结
tags: [Dnsmasq]
---

上一个星期在查找一个设备无法解析公司内部域名的问题，最终查出来问题是在网关设备上运行的 `Dnsmasq` 服务没有正确的配置上游 DNS 服务器，导致内网域名无法被正常的解析。在这期间对 `DNS协议` 又重新学习了一番，并且对 `Dnsmasq` 服务有了一些了解，结合网上一些资料，对 `Dnsmasq` 提供的 DNS 和 DHCP 服务的配置做一些总结和备忘。

`Dnsmasq`  是一个开源的项目，可以在 [thekelleys](http://www.thekelleys.org.uk/dnsmasq/doc.html) 上找到最新版本和源码，它能提供 DNS 、DHCP、TFTP、PXE 等功能。`Dnsmasq` 的 DNS 服务工作原理是，当接收到一个 DNS 请求是， `Dnsmasq` 首先会查找 `/etc/hosts` 文件，如果没有查找到，会查询本地 DNS 缓存记录，如果还是未找到对应的记录，则会将请求装发到 `/etc/resolv.conf` 文件中定义的上游 DNS 服务器中，从而实现对域名的解析。

基于上述原理，我们可以在 `/etc/hosts` 文件中添加本地内网的域名解析，从而实现本地内网的域名解析。同时我们还可以使用 `Dnsmasq` 来为一些特定的域名指定 DNS 服务器，或者阻止某些域名的访问。由于 `Dnsmasq` 会缓存上游 DNS 服务的查询记录，从而可以提高访问过的网址的连接速度。

默认情况下，`Dnsmasq` 会从 `/etc/dnsmasq.conf` 读取配置项，我们也可以使用 `-C` 的启动参数来指定配置文件。下面介绍一下常用的 DNS 和 DHCP 服务的配置参数。

<!--more-->

### 通用配置项

```
# 服务运行的网卡，如果有多个话，可在再次添加一条记录
interface=eth1
interface=wlan0

# 指定服务不在一下网卡上运行
except-interface=eth0

# 指定监听的 IP 地址，多个 IP 地址可用 `,` 分割(默认是监听所有网卡)
listen-address=192.168.8.132

# 开启日志选项，记录在 /var/log/debug 中
log-queries
  
# 指定日志文件的路径，路径必须存在，否则会导致服务启动失败
log-facility=/var/log/dnsmasq.log
 
# 异步log，缓解阻塞。
log-async=20
```

###  DNS 服务配置参数

```
# 指定 DNS 服务的端口，设置为 0 表示关闭 DNS 服务，只使用 DHCP 服务
port=63

# 指定一个 hosts 文件，默认是从 /etc/hosts 中获取
addn-hosts=/etc/banner_add_hosts

# 表示不使用 /etc/hosts 配置文件来解析域名
no-hosts

# 指定上游 DNS 服务列表的配置文件，默认是从  /etc/resolv.conf 中获取
resolv-file=/etc/dnsmasq.d/upstream_dns.conf

# 表示严格按照上游 DNS 服务列表一个一个查询，否则将请求发送到所有 DNS 服务器，使用响应最快的服务器的结果
strict-order

# 不使用上游 DNS 服务器的配置文件 /etc/resolv.conf 或者 resolv-file 选项
no-resolv

# 不允许 Dnsmasq 通过轮询 /etc/resolv.conf 或者其他文件来动态更新上游 DNS 服务列表
no-poll

# 表示对所有 server 发起查询请求，选择响应最快的服务器的结果
all-servers

# 指定 dnsmasq 默认查询的上游服务器
server=8.8.8.8
server=114.114.114.114

# 指定 .cn 的域名全部通过 114.114.114.114 这台国内DNS服务器来解析
server=/cn/114.114.114.114

# 给 *.apple.com 和 taobao.com 使用专用的 DNS
server=/taobao.com/223.5.5.5
server=/.apple.com/223.6.6.6

# 增加一个域名，强制解析到所指定的地址上，dns 欺骗
address=/taobao.com/127.0.0.1

# 设置DNS缓存大小(单位：DNS解析条数)
cache-size=500
```

`/etc/resolv.conf` 文件样例

```
nameserver 114.114.114.114
nameserver 8.8.8.8
```

`/etc/hosts` 文件样例

```
127.0.0.1         localhost 
192.168.101.107   web.gz.cvte.com web01
192.168.101.103   hrm.gz.cvte.com web02
```

### DHCP 服务配置参数

```
# 指定分配的 IP 端和续约时间
dhcp-range=192.168.1.50,192.168.1.100,12h

# 同上，指定了子网掩码
dhcp-range=192.168.8.50,192.168.8.150,255.255.255.0,12h

# 指定网关地址
dhcp-option=3,192.168.0.1

# 指定 DNS 服务器，net:eth1 用来指定网卡
dhcp-option=net:eth1,6,114.114.114.114，8.8.8.8
dhcp-option=net:wlano,6,114.114.114.114，8.8.8.8

# DHCP 所在的 domain
domain=gz.cvte.com

# 静态地址绑定
dhcp-host=00:0C:29:5E:F2:6F,192.168.1.201,os02
dhcp-host=00:0C:29:15:63:CF,192.168.1.202,os03

# 忽略一下 MAC 地址主机的请求
dhcp-host=11:22:33:44:55:66,ignore

# 租期保存文件
dhcp-leasefile=/var/lib/dnsmasq/dnsmasq.leases
```

dhcp-option 常用取值及含义

| option | option 作用 |
| :--: | :--: |
| 1 | 设置子网掩码选项 |
| 3 | 设置网关地址选项 |
| 6 | 设置DNS服务器地址选项 |
| 12 | 设置域名选项 |
| 15 | 设置域名后缀选项 |
| 33 | 设置静态路由选项。该选项中包含一组有分类静态路由（即目的地址的掩码固定为自然掩码，不能划分子网），客户端收到该选项后，将在路由表中添加这些静态路由。如果存在Option121，则忽略该选项 |
| 44 | 设置NetBios服务器选项 |
| 46 | 设置NetBios节点类型选项 |
| 50 | 设置请求IP选项 |
| 51 | 设置IP地址租约时间选项 |
| 52 | 设置Option附加选项 |
| 53 | 设置DHCP消息类型 |
| 54 | 设置服务器标识 |
| 55 | 设置请求参数列表选项。客户端利用该选项指明需要从服务器获取哪些网络配置参数。该选项内容为客户端请求的参数对应的选项值 |
| 58 | 设置续约T1时间，一般是租期时间的50% |
| 59 | 设置续约T2时间。一般是租期时间的87.5% |
| 60 | 设置厂商分类信息选项，用于标识DHCP客户端的类型和配置 |
| 61 | 设置客户端标识选项 |
| 66 | 设置TFTP服务器名选项，用来指定为客户端分配的TFTP服务器的域名 |
| 67 | 设置启动文件名选项，用来指定为客户端分配的启动文件名 |
| 77 | 设置用户类型标识 |
| 121 | 设置无分类路由选项。该选项中包含一组无分类静态路由（即目的地址的掩码为任意值，可以通过掩码来划分子网），客户端收到该选项后，将在路由表中添加这些静态路由 |
| 148 | EasyDeploy中Commander的IP地址 |
| 149 | SFTP和FTPS服务器的IP地址 |
| 150 | 设置TFTP服务器地址选项，指定为客户端分配的TFTP服务器的地址 |

> dhcp-option 遵循RFC 2132（Options and BOOTP Vendor Extensions)，可以通过 dnsmasq --help dhcp 来查看具体的配置很多高级的配置，如 iSCSI 连接配置等同样可以由 RFC 2132 定义的 dhcp-option 中给出。

### 参考资料

* [thekelleys 官网](http://www.thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html)
* [Dnsmasq使用参考入门](http://www.freeoa.net/osuport/servap/dnsmasq-use-intro-refer_2480.html)
* [dnsmasq (简体中文)](https://wiki.archlinux.org/index.php/Dnsmasq_%28%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87%29#DHCP_.E6.9C.8D.E5.8A.A1.E5.99.A8.E8.AE.BE.E7.BD.AE)