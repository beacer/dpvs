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

`DPVS`的实现在功能上保留了大部分`LVS`的特性，还引入了一些新的功能．实现过程参考了`LVS`及修改版本[alibaba/LVS](https://github.com/alibaba/LVS)，Kernel网络协议栈等，并利用`DPDK`等技术．成功实现多核线性扩展，成倍于`LVS`的性能提升．关键技术总结如下，

* **kernel-bypass**: 完全用户态实现，一是提升性能，二是方便功能开发、调试．
* **Share-nothing**: 关键数据*per-CPU*化，避免锁的使用（lockless）
* **避免上下文切换**：绑定网卡队列，处理线程和CPU，避免调度切换．
* **批处理化数据收发和处理**：批处理提高效率并采用*Run-to-Complete*模型．
* **Polling**：使用轮询而非内核的中断+下半部poll，专有CPU完全用于转发.
* **无锁化的消息系统**：用于高性能的跨CPU通信.
* 还有 *零拷贝*,*大页内存*，*prefetch* ...

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

其实LVS的性能瓶颈主要受限于Kernel，这么说也许许多人也许会很奇怪，毕竟Kernel一向给人以高水准，高性能的印象。而且多年来Kernel对网络部分的优化也从未停止，为何会成为性能瓶颈？ 首先，我们先看两组数据，一组是Google `Maglev`的，一组是`mTCP`的，这样对内核性能瓶颈有个直观的印象。

<center>
    <img src="pics/maglev-throughput.png" height="220"/>
    <img src="pics/mtcp-cps.png" height="220"/>
</center>

> 下方左边的图来自Google [Maglev的论文](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/44824.pdf)，右侧的图来自[mTCP的论文](https://www.usenix.org/system/files/conference/nsdi14/nsdi14-paper-jeong.pdf)。

不难看出，一方面在一个CPU core的情况下，基于kernel的方案在性能上就不如Bypass的方案；另一方面，现代SMP多CPU核的情况下Bypass的方案可以实现性能线性扩展，但是kernel却无法做到。

回到LVS，因为其主要部分ipvs是在内核态实现的，虽然有 *per-CPU连接表*, *缩小lock范围*，*尽早处理(pre_routeing)* 等优化手段，但来自kernel的性能问题依然存在。

### 为何Kernel成为性能瓶颈

那么为什么Kernel会成为性能的瓶颈？总结下来包括几个方面：

* 上下文切换的开销
* 锁的使用影响性能
* 大流量下中断过多（中断风暴）
* 复杂的协议栈导致路径过长

下面我们来以此说明一下。

##### 上下文切换

这个包括用户态内核态的切换，多进程/线程上下文的切换。上下文切换开销非常大应该尽量避免。所以各个application应该尽量将任务和CPU core进行亲核心绑定，就像Nginx的worker所做的那样。此外，不应该采用大量线程或进程的模型，应该让一个绑定了CPU的线程(进程)同时处理多个事务，这样也可以减少切换。同样可以参考Nginx的事件驱动（I/O多路复用+NONBLOCK_IO）的模型。

> 对应事件驱动模型，可以参考“C10K问题”以及具体的实现：Nginx，Lighttpd，libevent等。

##### 资源共享与锁的使用

我们知道，从Unix到Linux时分复用系统的基本目标就是提高资源利用率，使多个用户或者多个进程同时使用OS的资源，并且他们都以为自己独占了资源，多个进程/线程需要共享资源并进行切换，是并发的。而SMP的发展，更是让多个进程线程在物理上并行化。不管怎样，在有资源共享的情况下，一定需要保护资源，不然就会产生竞争条件。那么为了资源不会被竞争破坏只能加上锁（或原子变量，内存屏障）。

锁意味着，当拿不到资源的时候，只能等待（不论是切换出去等，还是忙等）都会造成CPU浪费，降低性能。SMP和进/线程多的情况下问题更加突出。虽然人们采用各种优化锁的技术，读写锁，spinlock，RCU，顺序锁...但是不管怎么锁法，都不如“没锁”来的高效。

但是Kernel资源共享的本质，却不可能没有锁的使用。就它的协议栈来说，虚拟设备需要锁，L2/L3/L4分用需要锁，Netfilter的hooks表需要锁，Conntrack需要锁，路由/ARP/IP地址需要锁，TCP/UDP层有锁，Socket层需要锁，TC(qdisc)需要锁...

Unix/Linux是“通用目的的操作系统”，它们从来就不是为单个进程，某个高性能场景服务的“专用设备”而生。设计之初也不是为SMP多CPU优化的。性能的提升和优化，只能是在资源共享、公平的前提下进行。比如缩小锁的范围，使用合适的锁，有些地方甚至可以per-CPU避免锁的使用，但终究太多资源共享了。毕竟不能破坏多用户、多任务，通用系统，公平这些大前提。因此，在某些特殊的高性能领域内Kernel协议栈表现的不够高效就比较好理解了。包括高速包转发，抗DDoS等。另外一方面，网卡的性能却逐渐提高，10G->25G->40G->100G不成为瓶颈。如果只是为了某个单一类型的应用，甚至硬件都可以是特定的，为什么还有跑一个完整的操作系统(协议栈)呢？

##### 中断模式
G
我们知道Kernel网卡驱动的收发包部分是中断（硬，软IRQ）和下半部实现的，通过NAPI接口实现了“中断加轮询”的方式，一方面利用中断来及时处理响应避免对用户造成延迟，另一方面为了兼顾吞吐量等性能，每次中断处理函数会以一定的配额去轮询(poll)设备的报文。这种机制，再结合各种硬件offload，和软件优化方案，能够最大程度兼顾大部分的延时和性能需求。

然而在高性能，高pps的特殊网络应用场景下，这种模式依然显得力不从心，当网络I/O（pps）非常大的时候可以看到CPU被大量消耗在软中断上。虽然可以使用中断/CPU亲核心设置充分利用网卡多队列和多core的能力，但中断本身还是会成为瓶颈的一部分。

DPDK的方案是轮询 。为什么会是“轮询”？ 一提轮询，大家首先想到间隔太短浪费CPU，间隔过长会造成不必要的延迟，怎么设置这个间隔都不好。但是，如果是不考虑CPU浪费，没有间隔，以“死循环”方式进行轮询，每次轮询尝试批量读取和处理数据，那么延迟问题能解决和性能也能大幅提升了。毕竟这个CPU全力在处理包的接收。

##### 强大也复杂的协议栈

即便每个函数都优化到极致、不消耗很多CPU，但是如果一个数据通过的函数调用链过长，其累计的性能消耗也会比较大。Kernel实现了太多的复杂的功能，Netfilter及相关的表和规则，ConnTrack系统，各种各样的QoS，GRO/TSO。为了封装不同协议、驱动等实现的抽象封装。这意味着一个数据包在内核中需要通过的函数非常的多，路径非常长，小的消耗积少成多。看看下图，Kernel网络栈的收发部分是不是特别的“高”？ 因为多层函数调用累计的消耗所占的比例就不算小了。在某些特殊的场景，其实不需要那么多的功能。

<center>
   <img src="pics/linux-perf-flame.png" width="520"/>
</center>

> 1. 图片来自ACM论文：https://queue.acm.org/detail.cfm?ref=rss&id=2927301
> 2. 同时也可以看到，用户态/内核直接的数据拷贝是否消耗CPU。不过LVS是内核态实现的，它没有这个问题。

##### 数据拷贝

> 首先说明，这点和LVS的性能可能关系没有那么大，因为内核态实现的ipvs，是不需要把数据copy到用户态再拷回去的，只是说数据拷贝非常影响性能。

网络数据通过内核和用户态边界的时候，不但有系统调用的开销，还需要一次额外的数据拷贝。如果能从网卡DMA到内存后，整个处理过程不存在额外的拷贝，也能提高性能。Linux的Sendfile机制就可以避免这种拷贝开销。而Kernel-bypass方案可以避免用户态/内核直接的拷贝，对于用户态转发数据而言会有两次拷贝。

# 如何提高性能

那么怎么怎么才能提高性能呢？ 经过上面的分析，我们知道传统LVS在高并发高流量下的性能瓶颈主要在Kernel。首先，简单直接的方法“Kernel-bypass”，绕过了Kernel既然也就没有了那些性能约束了。当然，事情并没有那么简单，我们从"Kernel-bypass"开始，结合其他的一些比较重要的技术，逐步分析如何综合利用他们来提高性能。

### 使用Kernel-bypass

###### 优点：高性能

使用Kernel-bypass技术可以避免kernel的瓶颈，解决上述提到的几个问题，带来成倍的性能提升。

除了性能之外，其他的好处包括：

###### 优点：用户态相对Kernel开发、和被采纳周期更短

  看看QUIC和TCP的改进就知道了，很多东西可以在QUIC上快速实验和应用，但是在TCP层实现意味着漫长的内核开发、采纳时间.另一个问题是无法强制用户去经常去升级Kernel，但升级一个app却十分常见。

###### 优点：用户态开发，调试更方便

  各种各样的调试和profiling手段，工具。程序挂了直接重启就行，解决coredump十分方便。虽然内核调试的技术也很多，但相对用户态而言显然更困难。何况招一个写应用的程序员，相对一个精通Kernel的程序员也方便的多（这主要是应用开发的需求比较多决定的）。

不过没有免费的午餐，我们还是要看看kernel-bypass面临的问题，

###### 缺点：太费事（too expensive）

  Kernel被bypass之后，协议栈也就没有了，我们需要在用户态重建TCP/IP协议栈，想象一下这里有多大的effert！而且重新实现一个协议栈，多少人有把握哪怕只是完成？更别说它能和Kernel一样稳定，能应付各种正常、异常情况，适用不同的场景保持高效和无bug？

  这里说明两点，

  1. 从0开始很难，但我们可以站在别人的肩膀上，事实上目前已经有不少各有特色的用户态协议栈了，其中有些是高性能的。seastar, mTCP, LKL, ODP/OFP, LwIP, f-stack, libuinet。比如f-stack，经过自研协议栈后最终还是决定使用了成熟的BSD栈（借鉴了libuinet），然后利用share-nothing思想，让每个CPU运行独立的stack，互不影响。这个做法比较巧妙，即带来客观的性能提升，又有成熟完善的协议栈支撑功能。再比如说LwIP其实更适合嵌入式系统，不太适合服务器。

  2. 有些应用场景其实并不需要完整的协议栈，比如说L2/L3数据转发，比如说4层负载均衡。4层负责均衡只需要4层端口信息（或者app的某个标识），并不需要实现完整的TCP，Socket层。只是实现3层功能（IP，ARP，Route，ICMP）虽然也很麻烦，但比起实现一个TCP或者Socket来说就好很多了。

###### 缺点：失去了多任务的能力

没有了内核协议栈，使用定制的协议栈保证高性能，我们也就同时失去了多任务的能力。网卡会被bypass技术（如DPDK）完全接管，普通的app看不到他们也无法直接使用，使用用户协议栈为了高性能而又隔离了app，他们可能无法在同一个系统上同时运行了。

> 网卡被某个DPDK app接管，就不能在它上面跑ssh这样的应用了，不过可以通过kni技术解决。但kni只是解决一部分问题，毕竟这部分(SSH)数据又通过了协议栈，kni做不到高性能。

不过，这对于单一目的的系统，比如一个服务器只用来做负载均衡器或Web Server就没有这个问题。这也是“通用”与”专用”的例子。

###### 缺点：许多Kernel配套功能用不了

比如ifconfig, ip没有了，tcpdump也很难真正应用，/proc文件没有了，现有的基于这些工具的profiling和监控、部署脚本，一整套配套的东西就无法使用了。更别谈iptalbes，和基于内核调参的优化了。这会给调试、排查问题和运维造成麻烦，还会增加工具、脚本等套设施开发的成本。是的，用户态应用本身的开发是变方便了，相关配套的成熟的东西却少了。

###### 缺点：稳定性、安全等

通过kernel-bypass，不谈工作量。新用户态协议栈稳定性（包括上述的f-stack, mtcp, seastar等）来看，它们显然没有经过Kernel那么多年的沉淀，而且有多少项目能有Kernel那么数量庞大又顶级的程序员参与，长期的高质量的保证（包括各个稳定发布版本）？除了稳定性，安全性如何保证，包括安全相关的功能和代码本身的安全性会不会有很多漏洞？这些也是问题。

###### Kernel-bypass的引用场景

分析了有缺点，就可以来看看它适合的应用场景了。总得来说包含两种场景：

* 高性能要求
* 低延迟要求

这样的场景包括

* 高频交易：需要超低延时，用户态定制的协议栈和polling模型可以保证低延时。
* 数据转发：这个属于高性能要求，包括2,3层交换、路由，和负载均衡器等。
* 抗DDoS：DDoS的大量的packet会在Kernel造成中断消耗过多CPU的问题，Kernel的pps能力不足。

可以看到其实数据转发和抗DDoS并不需要完整的协议栈支持，所以是应用kernel-bypass的合适场景。

> 当然Kernel也在不停的进化，比如说我们这里的提到的开源Ali/LVS，它的内核还是2.6.32，已经是非常“老”的Kernel了。Kernel对性能的追求从未止步过，多年前的"中断+轮询"，设备驱动napi/上下半部改造，RPS/LRO/GRO/..., REUSEPORT, 等等各种优化的引入，包括最近的XDP/eBFP方面的努力（也许正是针对DPDK这种kernel-bypass的威胁），facebook也开源了基于XDP的负载均衡器。还有不断进行中的各种TCP层，Socket层的优化，手段如此之多。所以说`Kernal-bypass`技术和Kernel自身的演化和对比应该是个动态的过程。

### Share-nothing思想
### 避免上下文切换
### 使用Polling而非中断
### 避免数据拷贝

网络数据通过内核和用户态边界的时候，不但有系统调用的开销，还需要一次额外的数据拷贝。如果能从网卡DMA到内存后，整个处理过程不存在额外的拷贝，也能提高性能。Linux的Sendfile机制就可以避免这种拷贝开销。而Kernel-bypass方案可以避免用户态/内核直接的拷贝，对于用户态转发数据而言会有两次拷贝。

> 当然内核态实现的ipvs，是不需要把数据copy到用户态的，只是说使用kernel-bypass,是可以实现zero-copy的。

参考资料
=======

* LVS (ipvs/ipvsadm/keepalived)
* Linux Network Stack
* Alibaba/LVS
* DPDK
* Kernel-bypass
  - https://blog.cloudflare.com/kernel-bypass/
  - https://blog.cloudflare.com/why-we-use-the-linux-kernels-tcp-stack/)
  - https://jvns.ca/blog/2016/06/30/why-do-we-use-the-linux-kernels-tcp-stack/
  - https://www.infoworld.com/article/3189664/networking/sdn-dilemma-linux-kernel-networking-vs-kernel-bypass.html
  - http://lukego.github.io/blog/2013/01/04/kernel-bypass-networking/
  - https://technologyevangelist.co/2017/12/05/kernel-bypass-security-bypass/
  - https://people.netfilter.org/hawk/presentations/LCA2015/net_stack_challenges_100G_LCA2015.pdf
