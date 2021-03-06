---
title: "软件工程实践之网段管理"
date: 2020-12-13T21:48:22+08:00
aliases:
  - /software-engineering-vpc
---

软件工程实践系列文章，
会着重讲述实际的工程项目中是如何协作开发软件的。
本文主要介绍了如何规划、搭建、管理网段。

<!--more-->

# outline

本文包括以下内容：

- concepts: 网段中的概念都很基础
  - ip/cidr: IP 是服务的点，而 CIDR 就是描述它的面
  - vpc/subnet: VPC 就是云服务提供划分网段的工具
- boundary: 网段的设计核心是边界划分
  - scene: 利用场景维度来划分，以区别开发、测试、生产环境
  - network: 利用网络维度来划分，以区分外网、内网
- example: 实际开发中的例子
  - ipsec: 用作 VPC 打通的常用工具
  - openvpn: 用作连接内网的常用工具
- conclusion


# Concepts

在我们日常开发中，
只要用到网络就一定会跟各类网段打交道。
很多时候网段作为一种稳定的基础设施，
我们并不会注意到它。
但是关于网段、网络的一些知识，
是确实在软件工程过程中需要考虑的。

## IP and CIDR

现代网络中，
IP 是服务的点，而 CIDR 就是描述它的面。

![ip][ip]

> IPv4 的类别分类

列举一些能想到的 IP，会包括：
`127.0.0.1` 这个约定的本地地址；
`8.8.8.8`/`8.8.4.4` 这个谷歌提供的 DNS 地址；
`59.78.3.25` 这个上海交大东三楼的 IP 地址…

而 CIDR 则是描述一系列 IP 地址的修饰符。

比如 `59.78.3.25/32` 代表二进制前 32 位都严格与 `59.78.3.25` 相等的 IP 地址段（也就是特指这个地址）。
类似地，`10.0.0.0/16` 就是以 `10.0.` 开头的 IP 地址段。


## VPC and subnet

AWS 作为云服务的始祖，
它开创了 VPC 服务让用户配置自己的网段，
后来 VPC 也逐渐成为了云服务的标配。

VPC 之间可以相互隔离，也可以打通。
（这里的隔离与打通，指的都是软件上的）
一般 VPC 会用作区分生产、开发环境。

比如我们可以在云服务上开两个 VPC，
生产环境的 CIDR 是 `10.0.0.0/16`，
开发环境的 CIDR 是 `10.1.0.0/16`。

而 VPC 下面可以拆分 subnet (子网)，
subnet 需要有更加具体的 CIDR，
比如生产环境的子网可能会是 `10.0.0.0/24`,`10.0.10.0/24` 这样的。


# Boundary

网段的核心是边界的划分。
而划分好网段的目的，
既有安全性的考量，
也会利用异地达到高可用容灾的目的。

## Scene

云服务一般在全球各大城市会设立数据中心，
每个数据中心会有多个机房，
我们网段的边界至少得以这个为界。

比如线上服务会跨机房部署，
分布在多个 subnet 中，
当某个机房发生清洁工拔电插吸尘器之类的事故时（一般不会发生）
服务能做到高可用。

subnet 级别还能配置网络流量的进出权限，
这点特性可以用来规划内部网络权限。

比如我们可以划分“外网可进可出”、“外网只出不能进”、“外网不可访问（仅可内网访问）”三类 subnet，
以部署不同特性的服务（负载均衡、普通业务、数据库）。

## Network

网络流量走的是内网还是外网也是值得考量的因素。

除了安全性因素以外，
一般的云服务商给外网流量定价 1元/GB，
价格因素也是一个需要考虑的点。

大部分时候，开发可以严格遵循分层的原则，
所有前端走的肯定是外网 API，
所有后端走的肯定是内网 API，
只有负载均衡、网关层考虑内外网转换的问题。

还有一些时候，可以通过 DNS 自动配置流量网络。
比如默认 `api.liriansu.com` 指向负载均衡的外网地址，
但是修改内网的 DNS 使其指向负载均衡的内网地址。


# Example

最后再附加一些实际开发中的例子。

## IPsec

在业务庞杂了以后，
就会遇到跨云服务商的开发场景。
跨云服务商的内网打通，
一般会用 IPsec 这个工具，
云服务自己提供的 VPN 服务也是基于此的。
IPsec 的本质是起两台能外网互通的服务器，
互相转发流量。

比如要配置阿里云与亚马逊云互通的步骤大概会包括：

- 两边搭建 VPC/subnet (CIDR 不重复)
- 两边各起一台有公网访问地址的机器，装好 ipsec
  - 配置 ipsec 证书、密钥等
  - 打开 ip forwarding, 配置好 iptables 防火墙规则
  - AWS 注意关掉机器的 source/dest check
- subnet 路由表配置跳转至 ipsec 机器
- 打开两边的安全组，即可互通访问
  - `traceroute` 命令可用于检测跳转

## OpenVPN

我们还会遇到的一种场景，
是开发环境接入内网开发。

这个一般可以用 OpenVPN 之类的工具打通，
但鉴于目前我们还是用的手动流量转发的路子去搞的，
等涉及这部分了再在之后的文章里分享吧 :)


# Conclusion

- 网段管理是跟开发息息相关的基建，最好能尽早规划好
- 网段规划的核心是边界划分，主要考虑安全性与高可用
- 云服务的基础概念是类似的，需要考虑好跨云服务、开发测试生产等多维度网段
- 内外网流量的复杂因素，可以通过分层或者 DNS 的方式屏蔽掉

（完）


[ip]: /assets/pics/ip_types.jpg

