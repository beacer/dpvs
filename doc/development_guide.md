DPVS开发者指南
============

# 目录

[第一部分：DPVS介绍及部署](#part-I)

* [什么是DPVS？](#whats-dpvs)
  - [什么是4层负载均衡](#whats-lb)
  - [项目由来](#prog-history)
  - [DPVS的特点](#dpvs-char)
* [试用DPVS](#try-dpvs)
  - [编译、运行](#build-run)
* [将DPVS用于生产环境](#product)
  - [转发模型](#fwd-model)
  - [调度模型](#shed-model)
  - [单臂与双臂模型](#one-two-arm)
  - [DPVS部署示例](#deploy)
    - [外网集群模式](#external-cluster)
    - [内网集群模式](#internal-cluster)
    - [主备模式](#master-backup)
    - [支持bonding和vlan](#bonding-vlan)
    - [SNAT集群](#snat-cluster)
    - [SNAT-GRE集群](#snat-gre-cluster)
  - [监控与运维](#monitor-admin)
    - [自动化部署](#auto-deploy)
    - [实现基本监控](#moitoring)

[第二部分：DPVS设计与实现](#port-II)

* [LVS为何无法实现高性能？](#why-lvs-not-quick-enough)
* [如何提高性能](#high-perf-howto)
  - [Kernel Bypass技术](#kernel-bypass)
  - [Share-nothing概念](#share-nothing)
  - [避免调度：绑定CPU/Queue](#cpu-affi)
  - [轮询和中断](#polling-interrupt)
  - [DPDK提供的相关技术](#dpdk-techs)
* [DPVS总体设计](#design-overview)
  - [设计目标](#design-goal)
  - [系统架构](#architect)
  - [模块划分](#modules)
  - [关键技术点](#key-tech)
  - [数据流(Big Picture)](#big-picture)
  - [选择与取舍](#consideration)
* [网络设备和数据收发](#netdev-rxtx)
  - [netif和数据收发](#netif)
  - [虚拟设备](#vdev)
    - [kni设备](#vdev-kni)
    - [vlan设备](#vdev-vlan)
    - [bonding设备](#vdev-bond)
    - [Tunnel设备](#vdev-tun)
    - [虚拟设备关系](#vdev-big-pic)
  - [tc: 流量控制](#tc)
    - [Linux的流量控制](#tc-linux)
    - [fifo: 先进先出队列](#tc-fifo)
    - [pfifo_fast: 优先级队列](#tc-pfifo-fast)
    - [tbf: 令牌桶过滤器](#tc-tbf)
    - [match: 规则匹配](#tc-match)
  - [硬件地址管理](#hw-addr)
* [用户态轻量级协议栈](#lite-ip-stack)
  - [公共API函数](#common-api)
  - [ipv4: 简单的IP层](#ipv4)
  - [neigh: ARP实现](#neigh)
  - [route: 路由实现](#route)
  - [inetaddr: IP地址管理](#inetaddr)
  - [icmp: 基本ICMP功能](#icmp)
  - [tunnel: IP-in-IP及GRE隧道](#tunnel)
* [负载均衡与转发](#ipvs)
  - [ipvs: 虚拟服务器](#ipvs-vs)
    - [ipvs基本概念](#ipvs-concept)
    - [ipvs核心数据流](#ipvs-core)
    - [返程数据亲和性](#traffic-back)
  - [xmit: 转发模型](#ipvs-xmit)
    - [FullNAT转发](#ipvs-xmit-fnat)
    - [DR转发](#ipvs-xmit-dr)
    - [SNAT模式](#ipvs-xmit-snat)
    - [NAT与隧道模式](#ipvs-xmit-nat-tunnel)
  - [sched: Real Server调度](#ipvs-sched)
  - [dest: Real Server管理](#ipvs-dest)
  - [service: 虚拟服务管理](#ipvs-service)
  - [proto: 协议转发](#ipvs-prot)
    - [TCP转发](#ipvs-prot-tcp)
    - [UDP转发](#ipvs-prot-udp)
    - [ICMP转发](#ipvs-prot-icmp)
    - [QUIC转发](#ipvs-prot-quic)
  - [laddr: Local IP管理](#ipvs-laddr)
  - [sapool: 地址池管理](#ipvs-sapool)
  - [toa/uoa: 获取真实客户端地址](#ipvs-toa-uoa)
  - [conhash: 一致性哈希](#ipvs-conhash)
  - [数据统计](#ipvs-stats)
* [其他基础部件](#basic-modules)
  - [timer: 高性能定时器](#timer)
  - [msg: 跨CPU无锁通信](#msg)
* [安全相关主题](#security)
  - [黑名单](#secu-blacklist)
  - [防syn flood攻击：syn-proxy](#secu-synproxy)
  - [并发、流量限制](#conn-traffic-limit)
* [DPVS控制面及工具](#control-plane)
  - [sockopt: 控制面](#sockopt)
  - [配置文件](#config-file)
  - [dpip工具](#dpip)
  - [ipvsadm工具](#ipvsadm)
  - [keepalived](#keepalived)
* [性能测试与调优](#perf-tune)
  - [性能测试](#perf-test)
  - [perf与火焰图](#perf-flamegragh)
  - [一步步改进性能](#perf-optimization)

[第三部分：开源合作与展望](#part-III)

* [开源的好处](#why-open-source)
* [DPVS开源的一些经验](#open-source-expirence)
* [持续集成](#ci)
* [自动化测试](#test-automation)
* [如何参与项目](#howto-contribute)
  - [提交bug](#report-bug)
  - [功能开发与Pull Request](#pull-request)
  - [提问与讨论](#ask-question-discuss)
* [项目的未来目标](#future)
* [参考资料](#references)

---------------------------------------------------------

<a id='part-I'/>

第一部分：DPVS介绍及部署
=====================

<a id='whats-dpvs'/>

# 什么是DPVS？

`DPVS`的名字来自与`DPDK`和`LVS`。`LVS`是多年来非常流行的4层负载均衡器，其核心转发部分在Linux内核态实现。而DPDK则是一个快速包处理的库和驱动套件，最早由Intel推出，现在已经成为Linux基金会项目，被广泛应用于各种领域．简而言之，`DPVS`就是“*基于DPDK的高性能4层负载均衡器*”。

> [LVS](http://linuxvirtualserver.org/)最早由大名鼎鼎的[章文嵩博士](http://jm.taobao.org/2016/06/02/zhangwensong-and-load-balance/)开发，随后包括百度，阿里在内的公司对它进行了改造，比如`FullNAT`和`syn-proxy`的引入，还有`SNAT`的patch；如今，随着DPDK等kernel-bypass技术的发展，特别适合对＂４层负载均衡＂这类纯转发设备的性能提升，于是各大公司也纷纷有了DPDK加速负载均衡的项目．

`DPVS`的实现在功能上保留了大部分`LVS`的特性，还引入了一些新的功能．实现过程参考了`LVS`及修改版本[alibaba/LVS](https://github.com/alibaba/LVS)，目前的Kernel网络协议栈等，并利用`DPDK`等技术．成功实现了成倍于`LVS`的性能提升，可实现多核线性扩展．相关的技术总结如下，

* **kernel-bypass**: 完全用户态实现，一是提升性能，二是方便功能开发、调试．
* **Share-nothing**: 关键数据*per-CPU*化，避免锁的使用（lockless）
* **避免上下文切换**：绑定网卡队列，处理线程和CPU，避免调度切换．
* **批处理化数据收发和处理**：批处理提高效率并采用*Run-to-Complete*模型．
* **Polling**：使用轮询而非内核的中断+下半部poll，专有CPU完全用于转发.
* **无锁化的消息系统**：用于高性能的跨CPU通信.
* 还有*零拷贝*,*大页内存*，*prefetch* ...

功能方面，`DPVS`则包括了`LVS`的特性和它所没有的一些功能．

* 多种负载均衡转发模式：`FullNAT`，`DR`，`Tunnel`, `NAT`模式．
* 各种*Real Server*调度算法: `RR`, `WLC`, `WRR`, 以及`一致性哈希`.
* 用户态轻量级高性能IP栈：实现了IPv4, ARP，ICMP，路由和地址管理等．
* 集成了`SNAT`模式：支持方向代理的同时、可用于IDC内部进行外网访问．
* 支持`kni`，`vlan`，`tunnel`，`bonding`等多种虚拟设备以适应复杂的IDC环境.
* `TOA`及新引入的`UOA`模块：用于`FullNAT`模式下`RS`获取真实TCP/UDP Client IP/Port．
* 安全和QoS相关：`syn-proxy`, 限并发，限流，黑名单, `tc`等模块．
* 工具支持：`dpip`工具，`keepalived`, `ipvsadm`，`quagga`等支持，用于部署监控。

---------------------------------------------------------

<a id='part-II'/>

第二部分：DPVS设计与实现
=====================

<a id='why-lvs-not-quick-enough'/>

# LVS为何无法实现高性能？

使用kernel-bypass技术加速4层负载均衡，Google应该是先行者，其论文告诉我们在高性能的场景下，kernel的瓶颈所在。
LVS是在内核态实现的，其性能问题也同样源自Kernel？


> 我们这里的对比对象是开源的LVS版本，它的内核还是2.6.32，已经是非常“老”的Kernel了。Kernel对性能的要求是非常高的，多年前的"中断+轮询"，设备驱动napi/上下半部改造，RPS/RFS/LRO/GRO/TSO/...等等各种方式的引入，包括最近的XDP/eBFP。还有各种TCP层，Socket层的优化。所以说`Kernal-bypass`技术和Kernel自身的演化和对比是个动态的过程。

所以kernel-bypass并非没有争论，

关于Kernel-bypass，几个有趣的文章不妨

https://www.infoworld.com/article/3189664/networking/sdn-dilemma-linux-kernel-networking-vs-kernel-bypass.html



---------------------------------------------------------

<a id='part-III'/>

第三部分：开源合作与展望
=====================
