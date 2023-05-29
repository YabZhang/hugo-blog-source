---
title: "DNS 解析配置文件 (On Linux)"
date: 2023-05-28T14:55:01+08:00
draft: false
categories:
    - linux
    - dns
tags:
    - hosts
    - resolv
    - nsswitch
---

- [1. /etc/hosts](#1-etchosts)
- [2. /etc/resolv.conf](#2-etcresolvconf)
- [3. /etc/nsswitch.conf](#3-etcnsswitchconf)
- [4. 总结](#4-总结)
- [5. 参考资料](#5-参考资料)

----


本文介绍三个 Linux 上的 DNS 相关的配置文件，分别是`/etc/hosts`, `/etc/resolv.conf` 和 `/etc/nsswitch.conf` 。

## 1. /etc/hosts

`/etc/hosts` 文件用于本地主机名解析，例如：

```bash
127.0.0.1	localhost
::1		    localhost ip6-localhost ip6-loopback
```

该文件最为常见，包含一组 IP 地址和主机名的映射关系。一般当主机名解析时，Linux 会首先在该文件中查找，如果找到对应的主机名，则直接返回对应的 IP 地址，否则才会去 DNS 服务器上查找。

## 2. /etc/resolv.conf

`/etc/resolv.conf` 文件用于配置 DNS 服务器，其格式如下：

```bash
// 本地主机配置
nameserver 192.xxx.xxx.xxx
options edns0
search .

// k8s 集群环境中的配置 
search      default.svc.cluster.local svc.cluster.local cluster.local
nameserver  10.96.0.10
options     ndots:5
```

`nameserver` 为配置的域名解析服务器地址，可以为 IPv4 或者 IPv6 地址；

`search` 指定了一组拓展搜索域；若域名中的 `.` 少于 `options ndots` 指定数目，则会以拓展搜索域轮流作为后缀尝试解析，直到得到一个解析结果；若全部拓展域都未匹配则返回报错。

例如：要在 k8s 环境解析应用域名 `webserver`，则会顺序解析 `webserver.default.svc.cluster.local`，`webserver.svc.cluster.local`, `webserver.cluster.local`，直到得到一个解析结果或最终解析失败。

`options` 为解析器配置项，如: `ndots`, `timeout`, `attempts`, `rotate`等。

具体可参见 Linux 的 [resolv.conf](https://man7.org/linux/man-pages/man5/resolv.conf.5.html) 文档。

## 3. /etc/nsswitch.conf

`/etc/nsswitch.conf` 文件用于配置系统的名称解析服务，其中 `hosts` 专用于域名解析，其格式如下：

```bash
hosts: files mdns4_minimal dns [NOTFOUND=return] myhostname
``` 

`files` 表示首先在 `/etc/hosts` 文件中查找；
`mdns4_minimal` 表示在 mDNS 服务器中查找，mDNS 服务器一般用于局域网中的主机名解析；

`dns` 表示在 DNS 服务器中查找；`[NOTFOUND=return]` 表示如果在前面的查找中未找到，则直接返回，不再继续查找；
`myhostname` 表示以自身主机名作为域名来解析，通常用于解析本地服务。

因此若任意的解析源得到了解析结果，就不会再继续查找后面的源。例如：若在 `/etc/hosts` 中找到了对应的 IP 地址，则不会再去后续 DNS 服务器中查找了。

## 4. 总结 

本文中，我们了解到三个 Linux 上域名解析相关的配置文件。其中：
* `/etc/hosts` 用于在文件中配置域名到 IP 的解析关系；
* `/etc/resolv.conf` 用于配置域名解析的具体参数，如: 服务器，拓展域等；
* `/etc/nsswitch.conf` 可配置各种名称解析，`hosts` 可配置域名解析源的相对顺序；

以上有助于促进对 Linux 上的域名解析机制的了解，更好地帮助我们定位域名解析相关问题。

## 5. 参考资料

1. https://man7.org/linux/man-pages/man5/resolv.conf.5.html
2. https://www.baeldung.com/linux/dns-resolv-conf-file
