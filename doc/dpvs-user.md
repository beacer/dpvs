DPVS用户手册
============

# 目录

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
* [DPVS部署示例](#deploy)
  - [外网集群模式](#external-cluster)
  - [内网集群模式](#internal-cluster)
  - [主备模式](#master-backup)
  - [支持bonding和vlan](#bonding-vlan)
  - [SNAT集群](#snat-cluster)
  - [SNAT-GRE集群](#snat-gre-cluster)
* [监控与运维](#monitor-admin)
  - [自动化部署](#auto-deploy)
  - [实现基本监控](#moitoring)

