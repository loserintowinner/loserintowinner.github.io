---
layout: post
title:  【网络】：虚拟 IP 原理
---

## **高可用性 HA（High Availability）**

　　指的是通过尽量缩短因日常维护操作（计划）和突发的系统崩溃（非计划）所导致的停机时间，以提高系统和应用的可用性。HA 系统是目前企业防止核心计算机系统因故障停机的最有效手段。

　　实现 HA 的方式，一般采用两台机器同时完成一项功能，比如数据库服务器，平常只有一台机器对外提供服务，另一台机器作为热备，当这台机器出现故障时，自动动态切换到另一台热备的机器。



## **怎么实现故障检测的那？**

​    心跳，采用定时发送一个数据包，如果机器多长时间没响应，就认为是发生故障，自动切换到热备的机器上去。



## 怎么实现自动切换那？

​    虚 IP。何为虚 IP 那，就是一个未分配给真实主机的 IP，也就是说对外提供数据库服务器的主机除了有一个真实 IP 外还有一个虚 IP，使用这两个 IP 中的 任意一个都可以连接到这台主机，所有项目中数据库链接一项配置的都是这个虚 IP，当服务器发生故障无法对外提供服务时，动态将这个虚 IP 切换到备用主机。

 



## 实现原理

　　主要是靠 TCP/IP 的 ARP 协议。因为 ip 地址只是一个逻辑 地址，在以太网中 MAC 地址才是真正用来进行数据传输的物理地址，每台主机中都有一个 ARP 高速缓存，存储同一个网络内的 IP 地址与 MAC 地址的对应关 系，以太网中的主机发送数据时会先从这个缓存中查询目标 IP 对应的 MAC 地址，会向这个 MAC 地址发送数据。操作系统会自动维护这个缓存。这就是整个实现 的关键。

　　下边就是我电脑上的 arp 缓存的内容。

```ini
(192.168.1.219) at 00:21:5A:DB:68:E8 [ether] on bond0
(192.168.1.217) at 00:21:5A:DB:68:E8 [ether] on bond0
(192.168.1.218) at 00:21:5A:DB:7F:C2 [ether] on bond0
```

　　192.168.1.217、192.168.1.218 是两台真实的电脑，

　　192.168.1.217 为对外提供数据库服务的主机。

　　192.168.1.218 为热备的机器。

　　192.168.1.219 为虚 IP。

　　大家注意 219、217 的 MAC 地址是相同的。

　　**再看看那 217 宕机后，我电脑上的的 arp 缓存**

```ini
(192.168.1.219) at 00:21:5A:DB:7F:C2 [ether] on bond0
(192.168.1.217) at 00:21:5A:DB:68:E8 [ether] on bond0
(192.168.1.218) at 00:21:5A:DB:7F:C2 [ether] on bond0
```

这就是奥妙所在。当 218 发现 217 宕机后会向网络发送一个 ARP 数据包，告诉所有主机 192.168.1.219 这个 IP 对应的 MAC 地址是 00:21:5A:DB:7F:C2，这样所有发送到 219 的数据包都会发送到 mac 地址为 00:21:5A:DB:7F:C2 的机器，也就是 218 的机器。





## 配置和删除虚拟 IP

假如主机有一个网卡 eth1，其对应一个 IP 为 192.168.1.217，现对其设置一个虚拟 IP 192.168.1.219：

```ini
ifconfig eth1:1 192.168.1.219 netmask 255.255.255.0
```

删除该虚拟 IP：

```ini
ip addr del 192.168.1.219 dev eth1
```

不过在网络运维中，更常见的是使用 keepalived 配置虚拟 ip（vip）实现双机热备以及自动切换主备，读者可以自行搜索研究。
