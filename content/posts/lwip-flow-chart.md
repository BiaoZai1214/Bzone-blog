---
title: "LWIP 流程结构框图"
date: 2025-12-03
draft: false
description: "图文并茂的LWIP精简TCP/IP协议栈流程框图"
tags: ["LWIP", "TCP/IP", "网络协议"]
cover: "/Bzone-blog/images/lwip-cover.png"
categories: ["知识库"]
---


这是一篇相对完善的 LWIP 流程结构知识框图，图文并茂，力求从一个初学者的角度，去理解嵌入式中最常用到的精简TCP/IP协议栈 -- LWIP。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531144026327.png)

> **注意 1**：本文大量参考《正点原子--LWIP网络协议栈》课程编写。
> **注意 2**：LWIP通常搭配RTOS一起使用，RTOS能够更方便管理网络线程任务。
> **注意 3**：LWIP 的协议栈过于庞大，知识量几乎是 RTOS 的 3-5 倍，所以细节部分也不如RTOS做得好。照我指导老师的话说：没有 5 年以上网络协议栈经验几乎不可能完全搞懂源码结构。**但对于初学者来说，懂流程和移植调用就已经足够了**。

因为绘图版式并不能在博客很好呈现，因此准备了绘图文件的在线版本与本地版本，使用浏览器或 Obsidian 可以查看获得最佳体验。

链接：

- 在线绘图： https://excalidraw.com/#json=xbY7ZXFpVj-fM1E-DGdX1,NB6GB0hheufQ3UEEGfHUDw

> 我会将本文开源到我的 GitHub 仓库和一些博客网站，同时对所有非商业的个人和企业完全开放，可以进行修改和教学参考，但我希望你能够保留我的冠名 -- 标仔。针对非商业，这是因为我反对把别人的开源精神拿去牟私利的行为，也许我的文章没有任何商业价值，但是，特此声明一下。
>
> 一个人的能力是有限的，希望大家能够和我一起共建一个对初学者更友好的平台。

---

## 一、物理层

### 1.1 TCP/IP 的分层模型

分层的本质就是：**专业的人干专业的事，互不干扰**。

教科书上讲的是 **OSI 七层模型**（理论完美但太复杂），实际嵌入式工程中用的是 **TCP/IP 五层模型**（我所知的互联网行业甚至是只分4层模型）

![](/Bzone-blog/images/lwip-flow-chart/file-20260531144709046.png)

从图中来看，发送和接收的流程就是，发送的时候层层打包，接收的时候层层拆包，你可以理解成就是一个收发包裹的过程。

| 层级 | 干什么 | 一句话类比 | 典型协议 |
|------|--------|-----------|----------|
| 物理层 | 电信号、介质特征 | 修路（铺网线、定义电平） | 以太网 PHY、Wi-Fi |
| 数据链路层 | MAC 地址、帧校验 | 快递员（按门牌号送包裹） | 以太网（802.3）、ARP |
| 网络层 | IP 地址、路由 | 导航（规划从 A 到 B 的路线） | IP、ICMP、IGMP |
| 传输层 | 端到端可靠传输 | 签收确认（保证包裹完整送达） | TCP、UDP |
| 应用层 | 具体业务数据 | 你写的信（具体内容） | HTTP、DNS、MQTT |

### 1.2 ETH 的框架图

这里展示了PHY到数据链路层的示意图：

![](/Bzone-blog/images/lwip-flow-chart/file-20260531154918325.png)

### 1.3 PHY 和 SMI 栈

网卡驱动层是 LWIP 和物理世界打交道的地方。在 STM32 上，你面对的硬件三件套是：

- **PHY 芯片**（比如 LAN8720A、YT8512C）：相当于"翻译官"，把 MAC 的数字信号翻译成网线上的电信号
- **MAC 控制器**（STM32 内置）：相当于"邮局"，负责帧的封装/解封装、CRC 校验
- **SMI 总线**（MDC + MDIO）：相当于"对讲机"，CPU 通过它读写 PHY 寄存器，获取链路状态

（以太网分了两大模式，RMII和MII。因为课程和例程都是基于LAN8720A芯片，所以只考虑RMII模式）

![](/Bzone-blog/images/lwip-flow-chart/file-20260531154937331.png)

#### 1.3.1 模式选择

在RMII下，根据芯片有无时钟晶振，分为了RMII1模式和RMII2模式（LAN8720A自带25MHz晶振）

![](/Bzone-blog/images/lwip-flow-chart/file-20260531154954587.png)

#### 1.3.2 LAN8720 特殊寄存器

![](/Bzone-blog/images/lwip-flow-chart/file-20260601133515610.png)

#### 1.3.3 BCR 和 BSR

PHY 寄存器遵循 IEEE 802.3 ，LAN8720A、YT8512C 这类 PHY 都兼容这套定义

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155003628.png)

---

## 二、DMA 描述符：让数据搬运不经过 CPU

DMA（Direct Memory Access）描述符是嵌入式以太网的"灵魂"。没有它，每收一个字节都得 CPU 亲自动手搬运，累死也处理不了几个包。

**DMA 描述符就像"快递单"：** 告诉 DMA 控制器"把这块数据从网卡搬到内存的这个地址，搬这么多字节，搬完后告诉我"。

### 2.1 DMA 发送描述符链的工作流程

1. CPU 准备好数据，填入描述符指向的发送缓冲区
2. CPU 设置描述符的 OWN 位（"这个包归 DMA 管了"）
3. DMA 控制器看到 OWN 位，自动把缓冲区的数据搬运到 MAC 控制器发出
4. 发送完成后，DMA 清除 OWN 位（"包发完了，缓冲区还给你"）
5. CPU 回收缓冲区，准备下一个包

### 2.2 DMA 接收描述符链的工作流程

1. DMA 从 MAC 接收数据，写入描述符指向的接收缓冲区
2. 接收完成后，DMA 设置 OWN 位（"数据放好了，你来处理"）
3. CPU 看到 OWN 位，从缓冲区读取数据并处理
4. 处理完后 CPU 清除 OWN 位（"缓冲区还给你，准备接下一个"）

**整个过程中 CPU 只负责"填单子"和"收快递"，搬运工作全部由 DMA 完成。** 这就是为什么嵌入式系统能处理高速网络数据的关键。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155106265.png)

#### 2.2.1 发送描述符成员

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155115784.png)

#### 2.2.2 接收描述符成员

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155123917.png)

### 2.3 环形单向链表

链表的结构，在《FreeRTOS源码结构框图》中也有做介绍。在以太网的DMA TX/RX描述符中也用到了类似的结构，以此来管理整个链路上的数据包。

**以太网 DMA 的 TX/RX 描述符，通过环形单向链表结构，让硬件可以自动循环遍历描述符节点，实现高效、持续的收发数据处理**。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155130151.png)

#### 2.3.1 追踪机制

**简单理解**：就像有 N 个收发"槽位"，硬件在这些槽位里循环工作，CPU 只需要处理标记为"完成"的槽位即可。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155139537.png)

#### 2.3.2 "接力赛机制"

这个"接力赛机制"就是让**以太网 DMA 自己按顺序、循环处理数据包**，CPU 只需要搭好"赛道"，剩下的搬运工作全交给硬件自动完成，实现高速低负载的收发。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155227928.png)

### 2.4 DMA 发送描述符链的构建流程

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155252275.png)

- **它是一个"翻译 + 打包"过程**：把应用层的原始数据，转换成以太网 DMA 硬件能识别的描述符链表格式。
- **DMA 只认描述符链**：它不认识你原始的链表结构，只认每个描述符里的 `buffer`、`len`、`next` 这三个字段。
- **CPU 只负责搭链，不负责搬运**：一旦把链交给 DMA，后续发送全是硬件自动完成，CPU 不用再逐个干预。

#### 2.4.1 发送描述符函数

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155307218.png)

### 2.5 DMA 描述符总结

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155319776.png)

### 2.6 LWIP 的内存管理策略

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155337365.png)

#### 2.6.1 内存堆的示意图

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155347713.png)

> LwIP 的内存堆，本质上是一块**连续的大内存池**，它支持你申请任意大小的内存块，用完后再归还回去。
> - 申请时：从内存堆里切一块合适的内存给你用；
> - 归还时：把这块内存放回堆里，和相邻的空闲块合并，避免碎片化

#### 2.6.2 内存堆初始化

#### （1）取出一块内存空间

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155401809.png)

#### （2）创建空闲块

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155408444.png)

#### （3）内存分配

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155415746.png)

#### （4）释放内存

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155422390.png)

#### （5）内存堆源码

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155438571.png)

#### 2.6.3 内存池简介

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155450186.png)

---

## 三、netif：LWIP 的"网卡身份证"

LWIP 用 `netif` 结构体来管理每个网络接口。如果你的 STM32 有两块网卡，就会有两个 `netif` 实例，通过 `next` 指针串成一个全局链表。

**为什么用链表？** 因为网卡数量不固定（可能 1 块，也可能 3 块），链表方便动态增删，不像数组需要预定义大小。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155502553.png)

### 3.1 netif 的关键字段

- **`num`**：索引号，从 0 开始，标识该网卡在链表中的顺序（第一个网卡 num=0，第二个 num=1）
- **`next`**：指向下一个网卡，最后一个为 NULL（链表尾部）
- **`state`**：网卡状态（比如是否 UP、是否有网线连接），由驱动维护
- **`input`**：接收入口函数指针 -- 网卡收到数据后调用它，把数据交给 LWIP 核心
- **`output`**：发送入口函数指针 -- LWIP 要发数据时调用它，把数据交给网卡驱动

### 3.2 netif 结构

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155515634.png)

### 3.3 网络接口 API

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155534650.png)

> lwIP 事件管理之消息邮箱实现：展示了 `sys_mbox` 如何基于 FreeRTOS 的 `xQueue` 实现消息传递，包括消息的发送（`sys_mbox_trypost`）和接收（`sys_mbox_fetch`）流程。

### 3.4 netif 的全局变量

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155548050.png)

> lwIP 线程模型：展示了 lwIP 的三个核心线程 -- 主线程（main thread）、tcpip_thread（TCP/IP 处理线程）、netif_thread（网卡管理线程），以及它们之间的消息队列通信方式。

### 3.5 全局链表操作流程

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155600792.png)

> **一句话理解 netif：它是 LWIP 和网卡驱动之间的"翻译官"，LWIP 不直接操作硬件，而是通过 netif 的函数指针间接调用驱动。** 这样换一块网卡只需要换一套驱动，LWIP 核心代码不用改。

---

## 四、超时事件与 pbuf

### 4.1 LWIP 超时事件管理

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155639856.png)

### 4.2 超时事件删除

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155708667.png)

### 4.3 超时事件查询

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155716937.png)

### 4.4 pbuf 是什么？

在普通程序里，你用 `malloc` 分配一块内存，往里面塞数据，用完 `free` 释放。但在网络协议栈里，数据包需要经过多层处理（加头部、查路由、校验等），如果每次都分配新内存、拷贝数据，CPU 会累死，内存也会碎片化。

**pbuf（Packet Buffer）就是 LWIP 解决这个问题的"杀手锏"。**

你可以把 pbuf 想象成一个**"快递箱"**：

- 箱子本身有固定大小的"管理信息"（pbuf 头部）
- 箱子里装着实际的数据（payload）
- 大的包裹可能需要多个箱子串联（pbuf 链）
- 每层处理时只需要移动"箱子里数据的起始位置"（指针偏移），不需要重新打包

**核心优势：数据只有一份，各层共享，零拷贝。**

> pbuf 结构体核心字段：`payload`（指向实际数据的指针）、`len`（当前 pbuf 中的数据长度）、`tot_len`（整条 pbuf 链的总长度）、`next`（指向下一个 pbuf）、`type_internal`（pbuf 类型：RAM/ROM/REF/POOL）、`flags`（标志位）。

#### 4.4.1 数据接收

![](/Bzone-blog/images/lwip-flow-chart/file-20260601164702313.png)

LWIP 设计了四种 pbuf 类型，各有各的用途：

| 类型 | 内存从哪来 | 优点 | 缺点 | 适用场景 |
|------|-----------|------|------|----------|
| `PBUF_RAM` | 从堆上 malloc | 灵活，大小随便定 | 慢，可能碎片化 | 发送数据（应用层写入） |
| `PBUF_ROM` | 指向 ROM/静态数据 | 零开销，不需要拷贝 | 只读，不能修改 | 发送不变的数据（如固定头部） |
| `PBUF_REF` | 指向外部 RAM | 零开销，引用已有数据 | 生命周期管理复杂 | 引用 DMA 缓冲区 |
| `PBUF_POOL` | 从预分配的池中取 | **极快**，O(1) 分配，无碎片 | 大小固定（通常 172 字节） | **接收数据**（高频操作） |

**为什么接收用 POOL？** 因为网卡中断每秒可能触发几千次，每次都要分配 pbuf。从预分配的池里"拿一个"比 `malloc` 快几十倍，而且绝对不会碎片化。

#### 4.4.2 链表结构

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155816125.png)

> pbuf 链结构示意图：展示了多个 pbuf 通过 next 指针串联的结构（Pbuf1 → Pbuf2 → Pbuf3 → PbufN，最后一个 next=NULL），每个 pbuf 包含 next、payload、tot_len、len、flags、ref 字段，右侧说明了各字段的含义。

#### 4.4.3 流程处理

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155826243.png)

> 数据使用流程：展示了应用程序/协议栈使用 pbuf 读取数据 → 处理数据 → 使用完后释放 pbuf → 归还内存池或释放内存的完整流程。

#### 4.4.4 pbuf 总结

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155845182.png)

### 4.5 pbuf 源码结构

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155900642.png)

> pbuf 分配函数实现代码：展示了 `pbuf_alloc` 函数的实现（根据 pbuf 类型选择不同的分配方式：PBUF_RAM 从堆分配、PBUF_POOL 从内存池分配、PBUF_REF/ROM 引用外部数据），以及 `pbuf_init_alloced_pbuf` 初始化函数和 `pbuf_alloc_reference` 引用分配函数。

#### 4.5.1 分配内存

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155908520.png)

#### 4.5.2 堆分配流程

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155917202.png)

---

## 五、IP 协议

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155930230.png)

### 5.1 MAC 地址和 IP 的区别

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155940096.png)

### 5.2 为什么需要 MAC 和 IP 地址

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155948715.png)

> 为什么要 MAC 地址和 IP 地址：展示了两个主机通过路由器通信的过程，解释了 MAC 地址用于同一局域网内的数据帧转发（逐跳改变），IP 地址用于端到端的路由选择（全程不变）。表格展示了数据在传输过程中 MAC 地址和 IP 地址的变化情况。

### 5.3 IP 的数据结构

![](/Bzone-blog/images/lwip-flow-chart/file-20260531155954818.png)

> IP 数据报结构（简化版）：展示了 IP 头部的 20 字节固定部分结构（版本 4 位、首部长度 4 位、服务类型 8 位、总长度 16 位、标识 16 位、标志 3 位、片偏移 13 位、TTL 8 位、协议 8 位、首部校验和 16 位、源 IP 地址 32 位、目的 IP 地址 32 位），以及 MTU（最大传输单元）= 1500 字节的概念。

### 5.4 如何分片

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160002286.png)

> IP 分片详解：展示了如何将 4000 字节的 IP 数据报分片（MTU=1500 字节，需要分 3 个分片），以及标志字段（Reserved 保留位、DF 不分片位、MF 更多分片位）和片偏移字段（Fragment Offset，以 8 字节为单位）的含义和计算方法。

### 5.5 分片原理

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160013484.png)

> IP 分片原理（简化版）：展示了 4000 字节数据报的分片过程 -- ①申请 pbuf 存储数据（4000 字节）→ ②交过传输层添加 TCP 首部（20 字节，总长 4020）→ ③交过网络层添加 IP 首部（20 字节，总长 4040）→ 按 MTU=1500 分成 3 个分片发送。

### 5.6 ip4_frag()

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160019753.png)

> IP 分片重组 ip_defrag 函数详解：左侧展示了分片到达过程（分片1/分片2/分片3），中间展示了重组流程（按 offset 排序、合并数据），右侧展示了重组完成后的完整数据报。底部说明了重组的注意事项。

### 5.7 IP 协议的重组

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160027230.png)

> IP 协议的重组（Reassembly）：展示了 IP 分片重组的过程，包括分片到达的乱序情况、重组表中的分片信息（序号、总长度、标志位、数据长度、说明），以及重组完成后的完整数据报。

### 5.8 接收端如何跟踪分片

![](/Bzone-blog/images/lwip-flow-chart/file-20260601191204407.png)

### 5.9 重组过程示意

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160041922.png)

> IP 分片重组过程示意图：左侧展示分片到达（乱序，分片2/分片1/分片3），中间展示按 offset 排序插入到对应的 pbuf 链（offset=0/185/370），右侧展示收到最后一个分片（MF=0）后合并为完整 IP 数据报（4000 字节）并交给上层协议。

### 5.10 起止位置保存

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160050145.png)

> 报文分片与重组辅助知识：左侧展示了标志字段（MF 位）和片偏移的含义，右侧解释了分片重组的概念，并特别说明了现代网络中应避免分片（通过 Path MTU Discovery 路径 MTU 发现机制），因为分片会降低网络效率。

---

## 六、ARP 协议

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160102259.png)

> ARP 协议（地址解析协议）详解：解释了 ARP 的作用（将 IP 地址映射为 MAC 地址），对比了 MAC 地址（用于局域网内数据帧转发，不可变）和 IP 地址（用于网络间路由选择，可变），说明了 ARP 协议在 TCP/IP 协议栈中的位置。

### 6.1 ARP 协议示意

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160111670.png)

> ARP 协议原理详解：展示了 ARP 的完整工作流程 -- IP 数据包到达后更新 ARP 缓存表，查找目标 MAC 地址，如果找到则直接发送数据包，如果找不到则发送 ARP 请求广播，等待 ARP 应答后缓存映射并发送数据包。

### 6.2 发送与接收 ARP

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160122843.png)

> ARP 数据结构和缓存管理：左侧展示了 ARP 的数据结构（arp_entry 结构体、ARP 缓存表、ARP 链表管理），右侧展示了裸机 APP 层和操作系统 APP 层如何直接管理 ARP 条目，包括 arp_input() 和 arp_output() 函数的调用关系。

### 6.3 ARP 的报文结构

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160136327.png)

> ARP 协议帧报文格式（数据链路层）：展示了 ARP 报文的完整结构，包括以太网首部（14 字节）、硬件类型、协议类型、硬件地址长度、协议地址长度、操作码（1=ARP 请求，2=ARP 应答）、发送方以太网地址、发送方 IP 地址、目标以太网地址、目标 IP 地址。

---

## 七、ICMP 协议

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160146928.png)

> 简单来说ICMP，是实现ip导通的助手，通常出现ping不通的问题，可能就是这里没开！

### 7.1 ICMP 协议类型与结构

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160156163.png)

### 7.2 ICMP 差错报文

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160205785.png)

### 7.3 PING 的过程

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160214131.png)

> PING 的过程示例：主机 A 发送 ICMP 回送请求（类型 8）→ 主机 B，主机 B 回复 ICMP 回送应答（类型 0）→ 主机 A。如果收不到应答，说明网络可能有问题。

---

## 八、TCP 协议

### 8.1 TCP 为什么这么复杂？

UDP 很简单：想发就发，发完就忘，不管对方收没收到。但 TCP 不行 -- 它要保证**数据完整、按序到达、不丢不重**。

怎么保证？靠一堆机制：序列号（标记每个字节的位置）、ACK 确认（告诉对方"我收到了多少"）、超时重传（没收到 ACK 就重发）、滑动窗口（控制发送速率）、拥塞控制（避免把网络堵死）......

这些机制加在一起，让 TCP 变成了一个"靠谱但话多"的协议。UDP 是"发了就跑"，TCP 是"发了还要等对方说收到，没说收到就再发一遍"。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160246716.png)

> LWIP 发送路径全景图：应用层 `send()` → TCP/UDP 封装 → IP 层处理 → ARP 查询 → 以太网帧封装 → 网卡驱动 `low_level_xmit()` → DMA 搬运 → MAC 发送 → PHY 转换为电信号。看到这张图你就知道一个包发出去要经过多少关卡。

### 8.2 完整的发送流程（以 TCP 为例）

1. **应用层**调用 `send()` 或 `write()`，数据进入 TCP 发送缓冲区
2. **TCP 层**：把数据拆成合适大小的段（MSS），加上 TCP 头部（源端口、目标端口、序列号、确认号、标志位等），调用 `tcp_output()` 发送
3. **IP 层**：`ip4_output_if_src()` 选择源 IP → 如果数据报超过 MTU 就调用 `ip4_frag()` 分片 → 加上 IP 头部（版本、源 IP、目标 IP、TTL 等）
4. **ARP 查询**：调用 `etharp_output()` 查 ARP 表，找到目标 IP 对应的 MAC 地址。如果 ARP 表里没有，就先发 ARP 请求广播，等对方回复后再继续发送
5. **以太网层**：`ethernet_output()` 加上以太网帧头（目标 MAC + 源 MAC + 类型），调用网卡驱动的 `output` 回调
6. **网卡驱动**：`low_level_xmit()` 把帧写入 DMA 发送描述符，设置 OWN 位，DMA 自动搬运到 MAC 发出

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160306641.png)

> TCP 首部结构：上方展示了代码形式的 `struct tcp_hdr` 结构体定义（包含 srcport、destport、seqno、ackno、flags、wnd 等字段），下方展示了 TCP 报文段首部的图示形式（32 位对齐的头部布局：源端口、目的端口、序列号、确认号、头部长度、保留位、标志位、窗口大小、校验和、紧急指针）。

### 8.3 接收路径：数据怎么从网线跑到应用层？

接收路径是发送的**逆过程**，数据从下往上走，每一层剥掉一个头部，最终还原为应用层数据。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160600415.png)

> LWIP 接收路径全景图：网卡收到帧 → `low_level_input()` 从 DMA 取数据 → `ethernet_input()` 解析帧类型 → ARP/IP 分发 → IP 层校验路由 → 传输层 TCP/UDP/ICMP 分发 → 应用层通过 `recv()` 读取。

### 8.4 完整的接收流程

1. **网卡收到帧**：PHY 收到电信号 → MAC 校验 CRC → DMA 搬运到接收缓冲区 → 触发以太网中断
2. **中断处理**：中断里不干活，只是发个信号量通知主任务"有数据来了"
3. **主任务处理**：等待信号量 → 调用 `ethernetif_input()` → 调用 `low_level_input()` 从 DMA 描述符链中取出 pbuf
4. **以太网层**：`ethernet_input()` 解析帧头的目标 MAC 地址，判断是否是给自己的（单播匹配、广播、组播），不是就丢弃
5. **协议分发**：根据帧类型字段分发 -- `0x0806` 是 ARP → `arp_input()`；`0x0800` 是 IP → `ip4_input()`
6. **IP 层**：`ip4_input()` 校验头部 checksum → 检查 TTL → 路由查找（判断是本机的还是需要转发）→ 本机的调 `ip_local_deliver()` → 如果有分片先调 `ip_defrag()` 重组
7. **传输层**：根据 IP 头部的协议字段分发 -- 6=TCP → `tcp_input()`；17=UDP → `udp_input()`；1=ICMP → `icmp_input()`
8. **应用层**：通过 `recv()` 或 `read()` 从 socket 读取数据

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160607832.png)

> UDP 数据传输流程（发送）：应用层数据 → UDP 数据（8 字节头 + 数据）→ IP 数据（20 字节头 + UDP 数据）→ 以太网帧（14 字节头 + IP 数据 + 4 字节 CRC）。展示了数据在每一层的封装过程。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160614752.png)

> 以太网层接收处理：`ethernet_input()` 解析帧头，检查目标 MAC 地址。如果是广播地址（FF:FF:FF:FF:FF:FF）或组播地址，直接处理；如果是单播地址，和自己的 netif MAC 比较，不匹配就丢弃。匹配的根据类型字段（0x0806=ARP, 0x0800=IP）分发。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160622293.png)

> ARP 接收处理：`arp_input()` 收到 ARP 请求后，如果目标 IP 是自己的就回复 ARP 应答（把自己的 MAC 塞进去），同时缓存对方的 IP-MAC 映射到 ARP 表。如果是 ARP 应答（别人回答了我的请求），就缓存映射。如果是无关的 ARP 包，直接丢弃。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160629440.png)

> IP 层接收处理流程图：`ip4_input()` → 校验头部 checksum（出错就丢弃）→ 检查 TTL（为 0 就丢弃并回复 ICMP 超时）→ 路由查找（`fib_lookup`，判断目标 IP 是否是本机地址）→ 本机的调 `ip_local_deliver()` → 如果有分片先调 `ip_defrag()` 重组 → 根据协议字段分发给传输层。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160634976.png)

> IP 分片重组：`ip_defrag()` 将收到的 IP 分片按标识符（ID）和片偏移量重新组装。所有分片到齐后才拼成完整的数据报交给传输层。如果某个分片超时未到，整个数据报被丢弃。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160650122.png)

> 传输层分发：IP 层根据协议字段将数据交给对应的处理函数。ICMP（ping 响应）走 `icmp_input()`，TCP（网页、SSH 等）走 `tcp_input()`，UDP（DNS、视频流等）走 `udp_input()`。三者独立处理，互不影响。

---

### 8.5 TCP 状态机

TCP 连接有 11 个状态，从 `CLOSED` 开始，经过一系列状态转换，最后回到 `CLOSED`。这张状态转换图是 TCP 的"灵魂"，面试必考，工作常用。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160317797.png)

> TCP 有限状态机：展示了 CLOSED、LISTEN、SYN_SENT、SYN_RECEIVED、ESTABLISHED、FIN_WAIT_1、FIN_WAIT_2、CLOSE_WAIT、CLOSING、LAST_ACK、TIME_WAIT 等 11 个状态之间的转换条件。箭头上的标注是触发转换的事件（SYN/FIN/ACK 等）和发送的报文。

#### 8.5.1 三次握手（建立连接）

> "喂，你在吗？在的，我在。好的，那我们开始吧。"

1. 客户端发 `SYN`（seq=x）→ 服务端（"我要连接你"）
2. 服务端回 `SYN + ACK`（seq=y, ack=x+1）→ 客户端（"收到，我也准备好了"）
3. 客户端发 `ACK`（seq=x+1, ack=y+1）→ 服务端（"好的，开始传数据"）
4. 双方进入 `ESTABLISHED` 状态

**为什么要三次而不是两次？** 因为如果只有两次，客户端发的第一个 SYN 如果在网络里"迷路"了（延迟到达），客户端超时重发第二个 SYN 并完成通信，然后关闭连接。但第一个 SYN 后来又到了服务端，服务端以为是新连接，白白分配资源等待 -- 这就是"历史 SYN"问题。三次握手通过第三步的 ACK 确认，避免了这种浪费。

#### 8.5.2 四次挥手（关闭连接）

> "我说完了。收到。我也说完了。好的，再见。"

1. 主动方发 `FIN` → 被动方（"我说完了"）
2. 被动方回 `ACK` → 主动方（"收到，但我还有话要说"）→ 半关闭状态
3. 被动方发 `FIN` → 主动方（"我也说完了"）
4. 主动方回 `ACK` → 被动方
5. 主动方等待 **2MSL**（Maximum Segment Lifetime，通常 60 秒）后关闭 → 确保最后一个 ACK 到达被动方

**为什么要四次而不是三次？** 因为 TCP 是全双工的，关闭连接需要双方各自关闭自己的发送通道。主动方发 FIN 只代表"我不发了"，但被动方可能还有数据要发，所以不能立刻跟着发 FIN，得等自己发完再说。

**为什么要等 2MSL？** 万一最后的 ACK 丢了，被动方会重发 FIN，主动方需要在 2MSL 内能够回应。另外，2MSL 保证本次连接的所有报文都在网络中消失，避免和新连接混淆。

### 8.6 TCP 的可靠性机制

TCP 靠三个核心机制保证可靠性：**确认（ACK）、重传、排序**。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160356237.png)

> TCP 数据传输基础：发送方维护发送窗口（Sliding Window），窗口内的数据可以连续发送而不需要等待每个包的 ACK。接收方通过 ACK 告诉发送方"我期望收到的下一个序列号是多少"。发送窗口随着 ACK 的到来不断右移。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160424770.png)

> TCP 三次握手建立连接：展示了客户端发送 SYN → 服务端回复 SYN+ACK → 客户端发送 ACK 的完整三次握手流程，以及每一步的序列号变化和状态转换。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160436180.png)

> TCP 四次挥手关闭连接：展示了主动方发送 FIN → 被动方回复 ACK → 被动方发送 FIN → 主动方回复 ACK 的完整四次挥手流程，以及 TIME_WAIT 状态的等待。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160500494.png)

> TCP 正常收发流程图（发送窗口=4）：展示了初始状态 → 发送 1-4 号数据 → 收到 ACK 确认 → 发送 5-8 号数据 → 收到 ACK 确认的完整流程，体现了滑动窗口的工作机制。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160526653.png)

> TCP 丢包重传机制：发送 1-4 号数据，2 号丢失 → 接收方收到 1、3、4 号但 2 号缺失 → 接收方返回重复 ACK → 发送方收到 3 个重复 ACK 后触发快速重传，立即重发 2 号数据。

#### 8.6.1 TCP 处理异常的四条黄金法则

1. **只有收到 ACK（累计确认）后，发送窗口才右移** -- 没确认的数据不能丢
2. **未收到 ACK → 超时重传对应数据** -- 超时了就再发一遍
3. **收到重复 ACK → 说明有数据未按序到达，重传缺失的数据** -- 快速重传，不等超时
4. **接收方只按序交付数据，并返回当前期望序号的 ACK** -- 乱序的先攒着，等齐了再交付

---

## 九、Socket API 与应用层协议

### 9.1 Socket：应用层的"万能插座"

Socket（套接字）就是应用层和传输层之间的**标准化接口**。你不需要自己处理 TCP 的三次握手、确认重传、流量控制这些破事，只需要调几个 API 就能收发数据。

**打个比方：** TCP 协议就像一条复杂的水管系统（有阀门、有压力表、有流量控制器），Socket 就是水管末端的"水龙头" -- 你只需要拧开水龙头（`socket()`）→ 接上水管（`bind()`）→ 打开总阀（`listen()`）→ 等人来接水（`accept()`）→ 用水（`send()`/`recv()`）→ 关水龙头（`close()`）。

**TCP 服务器端标准流程：**

```c
int server_fd = socket(AF_INET, SOCK_STREAM, 0);  // 1. 创建套接字
bind(server_fd, (struct sockaddr *)&addr, sizeof(addr));  // 2. 绑定 IP + 端口
listen(server_fd, 5);  // 3. 开始监听（backlog=5 表示最多排队 5 个连接）
int client_fd = accept(server_fd, NULL, NULL);  // 4. 等待客户端连接（阻塞）
recv(client_fd, buffer, sizeof(buffer), 0);  // 5. 接收数据
send(client_fd, response, strlen(response), 0);  // 6. 发送数据
close(client_fd);  // 7. 关闭连接
```

### 9.2 sockaddr：地址结构体的"前世今生"

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160726850.png)

> `sockaddr` 通用地址结构：`sa_family`（地址族，AF_INET=IPv4）+ `sa_data`（14 字节，IP 和端口混在一起）。问题是 sa_data 把端口和 IP 捆在一起，访问时得手动拆，很不方便。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160733779.png)

> `sockaddr_in` IPv4 专用结构：`sin_family`（AF_INET）+ `sin_port`（端口号，网络字节序）+ `sin_addr`（IP 地址，`in_addr` 结构体）+ `sin_zero`（8 字节填充，凑齐 16 字节）。各字段独立存储，操作方便。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160741323.png)

> SSI 动态网页示例：左侧展示了静态网页示例（无 SSI 命令），中间展示了 Web 服务器端处理流程，右侧展示了生成的动态响应内容（ADC_CH15 数值 2560，内部温度传感器 69.70°C）。

**实际使用口诀：** 定义用 `sockaddr_in`（方便填字段），传参强转成 `sockaddr *`（函数接口要求）。

### 9.3 HTTP：Web 的"通用语言"

HTTP（超文本传输协议）是基于 TCP 的应用层协议。你每天浏览网页、调 API，背后都是 HTTP 在工作。

**HTTP 的核心思想：** 客户端发一个"请求"（Request），服务器回一个"响应"（Response）。一来一回，完事。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160803047.png)

> Socket API 简介与常用接口说明：上方展示了 Socket API 的三层架构（SOCKET API → NETCONN API → RAW API），下方表格列出了 socket、bind、listen、accept、recv/read/recvfrom、write/send/sendto、close 等常用接口的功能说明。

**Web Server 工作流程（你写的嵌入式 Web 服务器也是这个逻辑）：**

1. 浏览器发 HTTP 请求：`GET /index.html HTTP/1.1\r\nHost: 192.168.1.100\r\n\r\n`
2. 你的服务器收到后解析请求行，知道对方想要 `/index.html` 这个文件
3. 打开 SD 卡上的 `index.html` 文件，读取内容
4. 构造 HTTP 响应头：`HTTP/1.1 200 OK\r\nContent-Type: text/html\r\nContent-Length: 1234\r\n\r\n`
5. 把响应头 + 文件内容一起通过 TCP 发回去
6. 浏览器收到后解析 HTML，渲染出网页

**嵌入式 Web Server 的关键配置：**

| 配置项 | 说明 | 常见坑 |
|--------|------|--------|
| 监听端口 | 默认 80（HTTP），建议用 8080 | 80 是特权端口，Linux 上需要 root 权限 |
| 根目录 | 文件存放路径 | 路径错了找不到文件，返回 404 |
| MIME 类型 | Content-Type 头部 | 类型错了浏览器不渲染（比如把 CSS 当 text/plain） |
| 并发处理 | 多客户端同时访问 | 嵌入式一般用单线程轮询或 FreeRTOS 多任务 |

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160813897.png)

> HTTP 响应报文结构：状态行（`HTTP/1.1 200 OK`）+ 响应头部（`Content-Type: text/html`, `Content-Length: 1234` 等，每行以 `\r\n` 结尾）+ 空行（`\r\n`，头部结束标志）+ 响应体（实际的 HTML/JSON/图片数据）。

### 9.4 MQTT：物联网的"微信群"

MQTT（Message Queuing Telemetry Transport）是一种**轻量级的发布/订阅模式**消息协议。如果 HTTP 是"打电话"（一对一，实时通信），MQTT 就是"微信群发消息"（一对多，异步通信）。

**MQTT 为什么适合物联网？**

- **轻量**：协议头只有 2 字节（HTTP 头部通常几百字节），适合低带宽网络
- **发布/订阅模式**：设备不直接通信，通过 Broker（代理服务器）中转。传感器只管发布数据，手机 App 只管订阅数据，互不依赖
- **支持 QoS**：有三个质量等级（0/1/2），可以根据可靠性需求选择
- **支持遗嘱消息**：设备异常断开时，Broker 自动发布预设消息通知其他设备

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160849950.png)

> MQTT 协议报文解析和交互流程：左侧介绍了 MQTT 协议基于 TCP/IP，报文由固定头 + 可变头 + 有效载荷组成；右侧展示了 MQTT 客户端与服务器之间的报文交互流程（CONNECT → CONNACK → PUBLISH → SUBSCRIBE 等）。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160856245.png)

> MQTT 发布/订阅模型：发布者（Publisher，比如温度传感器）将消息发送到 Broker（代理服务器，比如 EMQ X），订阅者（Subscriber，比如手机 App）从 Broker 接收感兴趣的主题消息。发布者和订阅者不直接通信，通过 Broker 解耦。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160904360.png)

> MQTT 报文结构：固定头部（2 字节）+ 可变头部（部分报文类型有）+ 有效载荷（实际数据）。固定头部的第一个字节是报文类型（4 位）+ 标志位（4 位），第二个字节是剩余长度（用变长编码，最大 4 字节可表示 256MB）。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160933979.png)

> MQTT 报文类型一览：CONNECT（客户端请求连接）、CONNACK（服务端确认连接）、PUBLISH（发布消息）、PUBACK（QoS 1 确认）、PUBREC/PUBREL/PUBCOMP（QoS 2 三步握手）、SUBSCRIBE（订阅主题）、SUBACK（订阅确认）、UNSUBSCRIBE（取消订阅）、PINGREQ/PINGRESP（心跳保活）、DISCONNECT（正常断开）。

#### 9.4.1 QoS 等级对比

![](/Bzone-blog/images/lwip-flow-chart/file-20260531160951424.png)

> MQTT QoS 等级对比：QoS 0（最多一次，fire and forget，可能丢消息）、QoS 1（至少一次，保证到达但可能重复）、QoS 2（恰好一次，最可靠但最慢，需要四步握手）。选择原则：不重要的数据用 QoS 0，重要数据用 QoS 1，金融级数据用 QoS 2。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161000272.png)

> MQTT QoS 0 流程（At most once）：发布者直接发送 PUBLISH 报文，不等待确认。最快最简单，但消息可能在网络中丢失。适合环境温度等不重要的周期性数据。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161007192.png)

> MQTT QoS 1 流程（At least once）：发布者发送 PUBLISH → 等待 PUBACK 确认 → 超时未收到就重传。保证消息至少到达一次，但可能重复（重传导致）。适合大多数 IoT 场景。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161030006.png)

> MQTT QoS 2 流程（Exactly once）：四步握手 -- PUBLISH → PUBREC（服务端收到）→ PUBREL（发布者确认释放）→ PUBCOMP（服务端完成）。保证消息恰好到达一次，不丢不重。开销最大，适合计费、控制指令等关键场景。

#### 9.4.2 主题与消息机制

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161037155.png)

> MQTT 主题（Topic）层级结构：用 `/` 分隔的层级字符串，如 `home/sensor/temperature`。支持通配符：`+`（单层通配，如 `home/+/temperature` 匹配 `home/sensor/temperature`）、`#`（多层通配，如 `home/#` 匹配 `home/` 下所有主题）。主题设计是 MQTT 系统架构的关键。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161044593.png)

> MQTT 保留消息（Retain Message）：发布者可以设置 RETAIN 标志，Broker 会保存该主题的最后一条保留消息。新订阅者订阅时，Broker 立即推送保留消息，不需要等下一次发布。适合"设备当前状态"这类需要立即获取的信息。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161108153.png)

> MQTT 遗嘱消息（Last Will and Testament）：客户端在 CONNECT 时预设遗嘱（主题 + 消息 + QoS + retain）。如果客户端异常断开（网络断线、电量耗尽等，未发送 DISCONNECT），Broker 自动发布遗嘱消息。其他订阅者收到后就知道"这个设备掉线了"。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161124409.png)

> MQTT 心跳机制（Keep Alive）：客户端在 CONNECT 时设置 Keep Alive 时间（单位秒）。客户端必须在该时间内至少发送一次 PINGREQ，Broker 回复 PINGRESP。如果超过 1.5 倍 Keep Alive 时间没有收到任何报文，Broker 认为客户端已断开，关闭连接并发布遗嘱消息。

> MQTT 以其简单、轻量、可靠、灵活的特点，成为物联网领域最常用的通信协议之一，特别适用于设备连接不稳定、网络带宽受限、设备资源有限的环境。

#### 9.4.3 连接与订阅流程

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161202478.png)

> MQTT 连接流程：客户端发送 CONNECT 报文（包含客户端 ID、用户名/密码、遗嘱消息、Clean Session 标志、Keep Alive 时间）→ Broker 检查认证信息 → 回复 CONNACK（返回连接接受/拒绝码，0x00=成功）。Clean Session=1 时 Broker 清除该客户端之前的会话信息。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161209121.png)

> MQTT 订阅流程：客户端发送 SUBSCRIBE 报文（包含主题过滤器 + QoS 等级，可以一次订阅多个主题）→ Broker 回复 SUBACK（返回每个主题授予的 QoS 等级，可以降级但不能升级）→ 之后该主题有消息发布时 Broker 自动推送给订阅者。

#### 9.4.4 MQTT 报文详解

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161227494.png)

> MQTT 协议报文介绍：左侧展示了 MQTT 报文的三种类型（连接报文、连接确认报文、发布报文、发布确认报文），右侧展示了固定头（Fixed header）的格式（报文类型 4 位 + 标志位 4 位 + 剩余长度）。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161236663.png)

> MQTT 报文类型表和可变头标志位：左侧展示了 MQTT 报文类型表（CONNECT、CONNACK、PUBLISH、PUBACK、SUBSCRIBE 等 14 种类型及其说明），右侧展示了报文头中可变头的标志位定义（第 4 字节 bit0-bit3 的含义）。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161244031.png)

> MQTT 可变头（Variable Header）详解：左侧展示了报文类型与可变头的对照表（不同报文类型的可变头内容不同），右侧以 PUBLISH 报文为例展示了可变头的具体结构（Topic Name 主题名 + Packet Identifier 报文标识符）。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161250687.png)

> MQTT 有效数据（Payload）详解：左侧展示了各报文类型的 Payload 内容（CONNECT 包含客户端标识符、遗嘱主题等，PUBLISH 包含主题名和消息内容等），右侧以 CONNECT 报文为例展示了 Payload 的具体结构（Client Identifier、Will Topic、Will Message、User Name、Password 等字段）。

#### 9.4.5 QoS 服务质量详解

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161300121.png)

> MQTT QoS 服务质量详解：介绍了 QoS 是 MQTT 协议中保证消息可靠性的核心机制，展示了三个级别的说明 -- QoS 0（最多一次，可能丢失，性能最高）、QoS 1（至少一次，不会丢失但可能重复）、QoS 2（恰好一次，不会丢失也不会重复）。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161307039.png)

> MQTT QoS 1 详解：展示了 QoS 1（至少一次）的消息传递流程 -- 发送者发送 PUBLISH（QoS=1）→ 接收者回复 PUBACK → 如果未收到确认则重发。特点：消息不会丢失，但可能重复（由重传引起），性能较好。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161313382.png)

> MQTT QoS 2 详解：展示了 QoS 2（仅一次）的四步握手流程 -- PUBLISH → PUBREC → PUBREL → PUBCOMP。特点：消息不会丢失且不重复，开销最大，适用于对数据一致性要求极高的场景（如支付、关键指令等）。

![](/Bzone-blog/images/lwip-flow-chart/file-20260531161325588.png)

> MQTT QoS 总结对比：QoS 0（最多一次，性能最佳但可能丢失）、QoS 1（至少一次，不会丢失但可能重复）、QoS 2（仅一次，不丢失不重复但开销最大）。选择建议：根据业务对可靠性和性能的要求选择合适的 QoS 等级。

---

---

到这里，LWIP 的流程结构就讲完了。让我们回到最初的分层模型来看，是不是更清晰了呢？

![](/Bzone-blog/images/lwip-flow-chart/file-20260601171051216.png)

**每个模块一句话总结：**

- **底层硬件**：PHY 翻译电信号，MAC 封装帧，DMA 搬运数据不经过 CPU，SMI 用来读写 PHY 寄存器
- **netif**：LWIP 和网卡驱动之间的"翻译官"，通过函数指针实现解耦
- **pbuf**：零拷贝的数据包管理，接收用 POOL（快），发送用 RAM（灵活）
- **发送路径**：应用层 → TCP/UDP → IP → ARP → 以太网帧 → 网卡发出
- **接收路径**：网卡 → 以太网 → IP → TCP/UDP/ICMP → 应用层
- **TCP**：靠序列号 + ACK + 重传保证可靠性，状态机有 11 个状态
- **Socket API**：应用层和传输层的"万能插座"
- **HTTP**：请求-响应模式，Web Server 就是一个监听 80/8080 端口的 TCP 服务器
- **MQTT**：发布/订阅模式，2 字节协议头，物联网的"微信群"

> **一句话总结：LWIP 就是"用分层架构 + 零拷贝 pbuf 实现了一套精简的 TCP/IP 协议栈"** -- 从网卡驱动到 Socket API，每一层各司其职，数据包在层与层之间流动时尽量不复制，只移动指针。

> **学 LWIP 的建议：** 先跑通例程（确保网卡能收发数据）→ 理解 pbuf 和数据流 → 搞懂 TCP 状态机 → 最后才是 MQTT/HTTP 这些应用层协议。不要一上来就啃源码，会劝退的。
