---
title: "iptables 笔记"
date: 2023-05-29T15:20:20+08:00
draft: false
categories:
    - linux
    - network
    - firewall
tags:
    - iptables
---

- [`iptables` 介绍](#iptables-介绍)
- [原理解析](#原理解析)
  - [1. 接收数据包](#1-接收数据包)
  - [2. `iptables` 原理](#2-iptables-原理)
- [总结](#总结)

---

# `iptables` 介绍

`iptables` 是一个用户空间实用程序，允许系统管理员配置 Linux 内核防火墙功能，如 IP 包过滤规则等，底层是基于 `netfilter` 内核模块实现 [^1]。 通过 `iptables` 可以对 Linux 系统的网络数据包执行检测、修改、转发、重定向和丢弃等操作, 应用层可以便捷高效地实现 帧过滤、网络审计、连接追踪、帧修改、NAT、Masquerading、端口转发和负载均衡等强大功能[^2]。很多知名项目都借助 `iptables` 实现完善功能，比如 `ufw`[^3], `docker`[^4], `HAProxy`[^5], `LVS`[^9] 等。

以下为一些常用的 `iptables` 命令[^6]：

```bash
# 通过 iptables-persistent 持久化保存 rules
# > sudo apt install iptables-persistent

# To accept all traffic on your loopback interface
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT

# allow outgoing traffic of all established connections
sudo iptables -A OUTPUT -m conntrack --ctstate ESTABLISHED -j ACCEPT

# allows established and related incoming traffic, allow return traffic for outgoing connections initiated by the server itself.
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# allow your internal to access the external (eth0 is your external network, and eth1 is your internal network)
sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT

# block connections originated from a specific IP address
sudo iptables -A INPUT -s 203.0.113.xx -j DROP

# reject the connection from the IP address, with a “connection refused” error
sudo iptables -A INPUT -s 203.0.113.xx -j REJECT

# allow all incoming SSH connections
sudo iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 22 -m conntrack --ctstate ESTABLISHED -j ACCEPT

# allow outgoing SSH connections
sudo iptables -A OUTPUT -p tcp --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p tcp --sport 22 -m conntrack --ctstate ESTABLISHED -j ACCEPT

# allow all incoming HTTP and HTTPS connections
sudo iptables -A INPUT -p tcp -m multiport --dports 80,443 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp -m multiport --dports 80,443 -m conntrack --ctstate ESTABLISHED -j ACCEPT

# allow incoming MySQL connections from a specific IP address or subnet
sudo iptables -A INPUT -p tcp -s 203.0.113.0/24 --dport 3306 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -p tcp --sport 3306 -m conntrack --ctstate ESTABLISHED -j ACCEPT
```

---

# 原理解析

那么 `iptables` 是如何实现这些功能的呢？

## 1. 接收数据包

当网卡接收到与本地 MAC 地址匹配的以太网帧或者是链路层广播，网卡设备会触发中断信号[^7]。

然后由网络驱动程序处理中断，并通过 `DMA` / `PIO` / 将数据包数据 (从 `RX Ring`) 读到内存中。在经过对数据包校验后，将数据帧内容存放到 skb(socket buffer) 待进一步处理。

`skb` 将被排入处理器中断队列中（并激活软中断，延迟执行[^8]）。如果队列积压已满，则在此处丢弃数据包。中断程序退出。此后 `do_softirq()` 将被调用，从队列中读出 `skb` 并将其传递给对应协议的处理函数 (IPv4 的情况下就是 IPv4 数据包处理程序了)。

在 IPv4 数据包处理程序中，首先进行一些检查和校验（如：IP校验和，长度和版本号等），然后将数据包传递给 `netfilter` 框架。


## 2. `iptables` 原理

`netfilter` 是一个 Linux 内核模块，它提供一个框架用于在数据包通过网络协议栈时执行操作。`netfilter` 通过 `netlink` 与用户空间通信，用户空间程序可以通过 `netlink` 接口向内核注册回调函数，当数据包通过网络协议栈时，内核会调用这些回调函数，从而实现对数据包的处理。

`netfilter` 提供五个 Hook 点分别是[^10]：

>
>**NF_IP_PRE_ROUTING**: This hook will be triggered by any incoming traffic very soon after entering the network stack. This hook is processed before any routing decisions have been made regarding where to send the packet.
>
>**NF_IP_LOCAL_IN**: This hook is triggered after an incoming packet has been routed if the packet is destined for the local system.
>
>**NF_IP_FORWARD**: This hook is triggered after an incoming packet has been routed if the packet is to be forwarded to another host.
>
>**NF_IP_LOCAL_OUT**: This hook is triggered by any locally created outbound traffic as soon as it hits the network stack.
>
>**NF_IP_POST_ROUTING**: This hook is triggered by any outgoing or forwarded traffic after routing has taken place and just before being sent out on the wire.

用户可以通过 `iptables` 命令向这些链中添加规则，当数据包通过网络协议栈时，内核会按照优先级依次根据这些规则来处理数据包。
`iptables`按照表来组织规则 ，常用的表为：`filter`, `NAT`(`SNAT` 和 `DNAT`), `mangle`, `raw` ； 每个表可包含多个规则链。
当一个数据包触发 Hook 点位时，内核会按照优先级依次执行规则链中的规则，直到遇到 `ACCEPT` 或 `DROP` 规则，或者规则链中的规则全部执行完毕。

详细的流转表[^11]：

![tables_traverse](https://www.frozentux.net/iptables-tutorial/images/tables_traverse.jpg)

# 总结
本文介绍了 `iptables` 的基本使用方法，以及基于 `netfilter` 内核模块的实现原理。整体上对 `iptables` 有了一般性的了解。
更深入的学习可以参考各种资料和开源项目，甚至可以自己动手实践。

[^1]: iptables Wiki: https://en.wikipedia.org/wiki/Iptables
[^2]: iptables Arch Wiki: https://wiki.archlinuxcn.org/wiki/Iptables
[^3]: ufw tutorials: https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-22-04
[^4]: docker(moby): https://github.com/moby/moby/blob/e1c92184f08153456ecbf5e302a851afd6f28e1c/libnetwork/iptables/iptables.go#L4
[^5]: HAProxy: https://github.com/haproxy/haproxy/blob/b7f8af3ca9424984e557f2c95c639dd4f57dfe61/doc/architecture.txt#L768
[^6]: iptables tutorails: https://www.digitalocean.com/community/tutorials/iptables-essentials-common-firewall-rules-and-commands
[^7]: The journey of a packet through the linux 2.4 network stack: http://ftp.gnumonks.org/pub/doc/packet-journey-2.4.html
[^8]: interrupts > Soft IRQs: https://linux-kernel-labs.github.io/refs/heads/master/lectures/interrupts.html#soft-irqs
[^9]: LVS tutorials: https://www.cnblogs.com/jmilkfan-fanguiju/p/10589739.html
[^10]: iptables&netfilter arch: https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture
[^11]: tables_traverse: https://www.frozentux.net/iptables-tutorial/iptables-tutorial.html#TRAVERSINGOFTABLES
