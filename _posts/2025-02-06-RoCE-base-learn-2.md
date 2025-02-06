---
title: "RoCE基础知识学习（二）"
date:   2025-02-06 15:51:00 +0800
categories:
  - ai
tags:
  - network
---

前面学习了 PFC 的基础知识，今天学习以下 PFC 的具体细节。

## 拾遗
前文提到的了一些点是错误的，这里进行修正。
- RoCE是如何实现无损网络的
  - 实际上无损网络是通过流量控制、拥塞控制、错误控制、特定协议等多种技术共同实现的，RoCE只是协议这一部分。

## 无损网络再理解
无损网络是为了达成以下四个目标：
- 不丢包 - 所有数据包都能到达目的地
  - PFC(Priority-based Flow Control)流量控制
  - ECN(Explicit Congestion Notification)拥塞通知
  - 缓冲区管理
  - 流量整形
- 不错序 - 数据包按发送顺序到达
  - 序列号机制
  - 重排序缓冲区
  - 有序投递机制
- 不损坏 - 数据内容保持完整性
  - 校验和(Checksum)
  - CRC校验
  - 错误检测码
- 不重复 - 每个数据包只交付一次
  - 序列号去重
  - ACK确认机制
  - 重传计时器
所以，想实现一个无损网络，需要各种机制配合完成，PFC+ECN只是解决不丢失这个目的的。

## RoCEv2的设计初衷
按照IB的官方架构文档说明，RoCEv2的设计目标是让IB流量跑在IP网络上以方便复用现有的网络设备和IP的路由能力。也就是说这并不是基于IP网络重新设计的协议，而是在IP网络上实现IB的流量。
- 为什么是UDP？
  目前并没有官方文档介绍这里的设计，可以做一些猜测：RoCEv2包含了IB的传输层，而UDP也是传输层协议，理论上IBTH不需要构建在UDP上，但是UDP的优势有以下几点，可以为RoCEv2提供一些便利：
  - UDP 有端口号，可以方便同一个IP地址上启多个RDMA应用
  - UDP 是一个成熟的协议，网络设备对UDP的支持比较好，如果RoCEv2直接构建在IP之上，需要网络设备支持新的协议号
  - UDP 是一个轻量级的传输层协议，工程友好
  > 官方提到使用UDP有一个额外的便利，就是在做ecmp的时候可以使用UDP的源端口进行哈希。

  由于RoCEv2的目的不是设计一个全新的协议，而是为了让IB跑在IP网络上，因此选择UDP是一个工程上非常实用的选择。

## RoCEv2的数据包格式
放一张官方的图：
![RoCE数据包格式](/assets/2025-02-06-RoCE-base-learn-2/pkg_header.png)
- 一些固定数据
  - RoCE v1: EtherType = 0x8915
  - RoCEv2:
    - EtherType = 0x0800 (IP)
    - IP Protocol = 17 (UDP)
    - UDP Port = 4791 (固定使用此端口号来标识RoCE流量)

看到报文结构可以发现，RoCE是基于链路层的，也就是说他无法使用三层交换机进行路由，所以RoCEv2比RoCE更加适合在数据中心内部使用。

## RoCEv2的报文要点
RoCEv2同时支持IPV4和IPV6
### IPV4
- IHL
  IHL要求设置为5，IP报文头如图所示：
  ![IP报文头](/assets/2025-02-06-RoCE-base-learn-2/ip_header.png)
  长度设置为5，意味着IP报文头长度为20字节(因为IHL的单位是32位也就是4字节)，Options字段为空，这是IP头的最小长度。
- ToS
  ToS字段比较关键，他的后两位是ECN字段，用来标识ECN是否开启，RoCEv2需要开启ECN，所以值应该是01或者10，如果不开启则设置为00。而DSCP字段用来标识RoCEv2的优先级是前6位，范围是0-63。一个比较常用的设置方法是：
  ![DSCP设置](/assets/2025-02-06-RoCE-base-learn-2/DSCP.png)
  而这个DSCP的值必须与RDMA的 Adress Vector 相对应，如果使用pytorch的RDMA初始化，那么DSCP的值应该使用环境变量设置`export RDMA_TRAFFIC_CLASS=46`。这样
- 其他字段
  - Total Length 将包含IP头部以及ICRC的长度
  - Flags 必须设置为"010"，意思是不分片
  - Fragment Offset 必须设置为0,因为不分片
  - TTL 需要跟 RDMA 的 HOP LIMIT 一致，可以`export RDMA_HOPLIMIT=64`
  - Protocol 设置为17（0x11），代表UDP
  - 源、目的IP必须跟RDMA的SGID index引用的地址一致

### IPV6
IPV6的设置原则基本与IPV4一致，不过实践中，大模型训练场景基本不会用IPV6，IPV4的地址是足够的，IPV6报文头又长，性能也不如IPV4。

### UDP
- 源端口：有时候会被用来做ecmp的策略，所以可以使用一些固定的源端口增强ecmp的效果
- 目的端口：大模型训练目的端口基本固定为4791，这样可以方便网络设备识别RoCE流量
- 长度：和IP头一样，需要包含ICRC
- 校验和：RoCEv2标准是设置为0

### ICRC
比较特殊的是RoCEv2中的ICRC会把可变字段当作'1'进行计算。包括：
- ipv4的 TTL,checksum,tos
- udp的checksum