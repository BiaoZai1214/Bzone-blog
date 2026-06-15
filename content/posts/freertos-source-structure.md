---
title: "FreeRTOS源码结构框图"
date: 2025-04-13
draft: false
description: "图文并茂的FreeRTOS源码结构知识框图"
tags: ["FreeRTOS", "RTOS", "源码"]
cover: "/Bzone-blog/images/rtos-cover.png"
categories: ["知识库"]
---

这是一篇相对完善的 FreeRTOS 源码结构知识框图，图文并茂，力求从一个初学者的角度，去理解嵌入式中最常用到的操作系统 -- FreeRTOS。

> ⚠️：本文主要是对**源码结构**的解读，不涉及调用 API 函数的教学。

![](/Bzone-blog/images/freertos-source-structure/file-20260529222434170.png)

因为绘图版式并不能在博客很好呈现，因此准备了绘图文件的在线版本与本地版本，使用浏览器或 Obsidian 可以查看获得最佳体验。

链接：

- 在线绘图： https://excalidraw.com/#json=z5cODpq9y9JDqXRfgxMyH,u54TebroibvCocW71_fWdQ

> 我会将本文开源到我的 GitHub 仓库和一些博客网站，同时对所有非商业的个人和企业完全开放，可以进行修改和教学参考，但我希望你能够保留我的冠名 -- 标仔。针对非商业，这是因为我反对把别人的开源精神拿去牟私利的行为，也许我的文章没有任何商业价值，但是，特此声明一下。
>
> 一个人的能力是有限的，希望大家能够和我一起共建一个对初学者更友好的平台。

---

## 一、链表

### 1.1 普通链表

**链表**是一种**线性数据结构**，和数组类似，都用来顺序存储数据，支持"增删改查"。

简单理解：数组是一排**连续的格子**，按下标直接找到；链表是**散落的格子**，靠一张纸条（指针）告诉你下一个格子在哪。

- **数据域**：真正要保存的内容
- **指针域**：指向下一个节点的地址

### 1.2 哨兵节点

哨兵节点就是个**占位符**，代表"这里有个链表"，不管它有没有实际数据。普通链表里它通常是第一个节点，也叫头结点。

**为什么要哨兵？** 没有它的话，插入第一个节点得特殊处理（头指针为空的情况），有了它，所有操作都能统一处理，不用写一堆 if 判断。

![](/Bzone-blog/images/freertos-source-structure/file-20260529184840197.png)

> 这里展示了 3 种普通链表的结构示意图，笔者留下了这几个链表的 C 语言源代码实现，提供参考。

### 1.3 FreeRTOS 链表结构

RTOS 的链表和普通链表类似，但有一个关键区别：**哨兵节点放在最后面**，用最大值 `portMAX_DELAY`（`0xFFFFFFFF`）当排序值，保证它永远在末尾。

FreeRTOS 用了两个核心结构体：

- **`List_t`**：链表本体，记录链表长度、指向哨兵的指针这些
- **`ListItem_t`**：链表节点，每个节点带着排序值 `xItemValue`，还有前驱/后继指针和拥有者指针（通常指向 TCB）

**排序规则：** 链表按 `xItemValue` 升序排列，插入时从头遍历，找到第一个比新节点大的位置，插在它前面。哨兵的值是最大值，所以它永远不会被超过，天然卡在末尾。

![](/Bzone-blog/images/freertos-source-structure/file-20260529191858708.png)

#### （1）初始化

`vListInitialise()` 把 `List_t` 里的所有指针都指向哨兵节点自身 -- 因为链表是空的，只有它一个。

![](/Bzone-blog/images/freertos-source-structure/file-20260529192020629.png)

#### （2）插入链表

`vListInsert()` 按升序把新节点插入，本质就是**双向链表的插入**：改新节点的前后指针，再改前后节点的指针指向它。每插一个，`uxNumberOfItems` 加 1。

![](/Bzone-blog/images/freertos-source-structure/file-20260529192152599.png)

#### （3）移除链表

`uxListRemove()` 把节点摘掉，只需要让它的前一个和后一个互相指就行。摘完 `uxNumberOfItems` 减 1。

![](/Bzone-blog/images/freertos-source-structure/file-20260529192253567.png)

> **Q：为什么要用尾哨兵，不用首部哨兵？**
>
> A：RTOS 链表是升序排列的，所有有效节点数值从小到大排。
>
> 1. 用首部哨兵的话，得设成最小值，但新节点会不断排到它后面，哨兵就固定不住了，还得加分支判断。
> 2. 用尾哨兵设成最大值，所有有效节点都比它小，哨兵自然就卡在最后。
> 3. 这样插入、遍历、删除都用一套逻辑，不用区分头尾边界，代码更简洁，也不容易出空指针问题。

---

## 二、任务创建与调度

### 2.1 静态任务与动态任务

FreeRTOS 创建任务有两条路：**静态创建**和**动态创建**。流程几乎一样，区别就是内存谁来管：

| | 静态创建 | 动态创建 |
|---|---|---|
| TCB 和栈 | 你自己手动分配 | FreeRTOS 自动 `pvPortMalloc()` |
| 释放内存 | 你手动释放 | `vTaskDelete()` 时自动释放 |
| 适用场景 | 不允许动态内存的系统 | 大多数通用场景 |

因为静态创建就比动态创建多了两步手动操作，所以掌握静态的流程，动态的也就懂了。

![](/Bzone-blog/images/freertos-source-structure/file-20260530125641308.png)

**静态创建的两步手动操作：**

1. 定义一个 `StaticTask_t` 变量当 TCB
2. 定义一个 `StackType_t` 数组当任务栈（大小由 `uxStackDepth` 决定）

![](/Bzone-blog/images/freertos-source-structure/file-20260530161039110.png)

### 2.2 TCB 结构体

不管动态还是静态创建的任务，操作系统都会创建一个 TCB（Task Control Block）结构体。**TCB 里面自带链表节点，内核就是靠这些节点把 TCB 串成链表来管理任务的。**

**TCB 里比较重要的字段：**

- **`pxTopOfStack`**：栈顶指针，任务切换时从这里恢复/保存寄存器
- **`pxStack`**：栈的起始地址，用来释放内存和检测栈溢出
- **`xStateListItem`**：状态链表节点，挂到就绪列表/阻塞列表/挂起列表用的
- **`xEventListItem`**：事件链表节点，挂到队列/信号量的等待列表用的
- **`uxPriority`**：优先级，数值越大越高（默认 0 最低）
- **`pcTaskName`**：任务名字，调试用的

![](/Bzone-blog/images/freertos-source-structure/file-20260530130502515.png)

### 2.3 静态任务的创建流程

前面是表层 API，现在要往源码深处走了。

`xTaskCreateStatic()` 内部就两步：

![](/Bzone-blog/images/freertos-source-structure/file-20260530132434149.png)

**你需要记住的要点：**

1. **`prvInitialiseNewTask`** -- **填充 TCB 数据**
   - 把 TCB 清零
   - 设置名字、优先级、栈深度
   - 算出栈顶指针，指向栈数组末尾
   - **在空栈上伪造一份"刚被中断打断过"的硬件栈帧**（最关键的一步）
   - 初始化 TCB 里的状态节点和事件节点

2. **`prvAddNewTaskToReadyList`** -- **把 TCB 插入就绪链表**
   - 关中断（临界区保护）
   - 把任务插入到对应优先级的就绪链表里
   - 全局任务计数加 1
   - 如果新任务优先级比当前运行的高，触发一次抢占

数据填好了，任务也放进就绪列表了。

**伪造栈帧是什么意思？**

就是在空栈上手工写一份"假装之前正在运行、被 PendSV 中断切走了"的寄存器数据。这样 CPU 第一次调度这个任务时，硬件会自动恢复这些寄存器，任务就从指定的入口函数开始跑了。

`portInitialiseStack()` 在栈上从高地址到低地址依次压入：

- **xPSR**：`0x01000000`（Thumb 模式位）
- **PC**：任务函数入口地址（恢复后 CPU 从这里开始执行）
- **LR**：`prvTaskExitError`（任务意外返回时的兜底处理）
- **R12, R3, R2, R1**：0
- **R0**：任务参数 `pvParameters`
- **R11-R4**：0

![](/Bzone-blog/images/freertos-source-structure/file-20260530151747173.png)

### 2.4 启动调度器

资源都分配好了，该启动调度器了。

调度器就是 **FreeRTOS 的"灵魂"**，负责决定哪个任务该上 CPU、哪个任务该靠边。

![](/Bzone-blog/images/freertos-source-structure/file-20260530154500895.png)

`vTaskStartScheduler()` 内部做了四件事：

1. 创建一个**空闲任务**（优先级最低，保证任何时候都有任务可跑）
2. 调用 `xPortStartScheduler()` 进入硬件相关的端口层
3. 把 **PendSV** 和 **SysTick** 中断优先级设为最低（不让任务切换阻塞其他中断）
4. 调用 `prvStartFirstTask()` 触发第一次切换

#### （1）预启动

`prvStartFirstTask` 不直接切换，它只是**把环境准备好，然后通过 SVC 中断把接力棒交给 `vPortSVCHandler`**。

它做了两件事：
- 设置 MSP（主栈指针）初始值
- 执行 `svc 0` 触发 SVC 异常

![](/Bzone-blog/images/freertos-source-structure/file-20260530160305818.png)

#### （2）触发 SVC 切换

`vPortSVCHandler` 才是真正干活的：

- 从就绪列表找到最高优先级任务的 TCB
- 把 TCB 里的 `pxTopOfStack` 加载到 PSP（进程栈指针）
- 设置 LR 为 `0xFFFFFFFD`，告诉 CPU 异常返回后用**线程模式 + PSP 栈**
- 执行 `bx lr`，CPU 自动从 PSP 栈恢复寄存器，开始跑第一个任务

![](/Bzone-blog/images/freertos-source-structure/file-20260530160559327.png)

**补充说明：**

1. **`cpsie f` 指令**：Cortex-M 大部分内核不支持 FIQ，这条指令在 M3/M4/M7 上无效，只是为了兼容。
2. **LR 的 `0xFFFFFFFD`**：意思是异常返回后用线程模式 + PSP 栈，这是任务切换的关键。
3. **`vTaskDelete(NULL)`**：启动任务完成创建后就"自杀"了，不再占 CPU，只让业务任务跑。

### 2.5 就绪列表

就绪列表是个 **`List_t` 类型的数组**，下标就是优先级。比如 `pxReadyTasksLists[3]` 就是优先级为 3 的就绪链表。

- 每个优先级对应一个链表，里面挂着该优先级下所有就绪的任务
- 调度器从下标 0 开始找，第一个非空链表就是最高优先级的任务
- 同优先级的任务按创建顺序排列，靠时间片轮转
- 任务创建、恢复、从阻塞唤醒时 → 插入就绪链表
- 任务阻塞、挂起、被抢占时 → 从就绪链表移除

### 2.6 运行态与任务切换

调度器从就绪列表挑出**最高优先级的任务**，把 CPU 给它，任务就进入运行态。同一时刻只有一个任务在跑（单核），FreeRTOS 用**抢占式调度**：高优先级任务可以随时打断低优先级任务。

任务离开运行态有三种方式：

1. **主动让出**：调 `vTaskDelay()` 或等信号量/队列 → 进阻塞状态
2. **被抢占**：更高优先级任务就绪了 → 调度器在 SysTick 或 PendSV 中断里切换
3. **被挂起**：调 `vTaskSuspend()` → 进挂起状态

任务切换靠 **PendSV 中断**（优先级最低的可悬起异常），流程是：

1. **保存当前任务**：把 R4-R11 压入当前任务的栈，更新 `pxTopOfStack`
2. **切换指针**：`pxCurrentTCB` 指向新任务的 TCB
3. **恢复新任务**：从新 TCB 的 `pxTopOfStack` 取栈顶指针加载到 PSP，弹出 R4-R11
4. **异常返回**：`bx lr`（`EXC_RETURN = 0xFFFFFFFD`），CPU 自动恢复 R0-R3, R12, LR, PC, xPSR

当前任务把寄存器**保存到自己的堆栈**里 → **入栈**；下次轮到时再**恢复寄存器** → **出栈**。

![](/Bzone-blog/images/freertos-source-structure/file-20260530161509475.png)

![](/Bzone-blog/images/freertos-source-structure/file-20260530162247809.png)

### 2.7 阻塞与延时链表

> **"阻塞列表"和"延时链表"不是两套东西，而是同一回事的两个名字。**

"阻塞"描述的是任务的状态（因为等某个事件所以跑不了），"延时链表"描述的是这个状态在源码里的实现（`pxDelayedTaskList`）。

不管你是 `vTaskDelay()` 延时，还是 `xSemaphoreTake()` 等信号量，还是 `xQueueReceive()` 等队列数据，底层都是调同一个函数 `vTaskPlaceOnEventList()`：算出唤醒时间，按升序插到延时链表里。

**两个延时链表解决溢出问题：**

`xTickCount` 是 32 位变量，加到 `0xFFFFFFFF` 后溢出变成 `0`。如果只有一个链表，溢出后内核会误判任务已到期。FreeRTOS 维护两个链表（`xDelayedTaskList1` 和 `xDelayedTaskList2`），一个存"本周期到期"的，一个存"跨周期到期"的。溢出时指针互换，跨周期的就变成本周期的了。

![](/Bzone-blog/images/freertos-source-structure/file-20260530163128781.png)

### 2.8 挂起列表

挂起列表就是个**简单单链表**，存放 `vTaskSuspend()` 挂起的任务。

| | 阻塞 | 挂起 |
|---|---|---|
| 怎么触发的 | 等事件（延时、信号量等） | 手动调 `vTaskSuspend()` |
| 怎么恢复 | 事件到了自动回就绪列表 | 必须手动调 `vTaskResume()` |
| 能响应中断吗 | 不能 | 不能 |
| 参与调度吗 | 不参与 | 不参与 |

> **一句话串起整个流程：** 创建任务 → 填充 TCB → 放入就绪列表 → 启动调度器 → 调度器选出最高优先级任务 → 进入运行态；任务因延时/等待/挂起离开运行态，进入阻塞/挂起列表；条件满足后回到就绪列表，等待下一次调度。

![](/Bzone-blog/images/freertos-source-structure/file-20260530191616821.png)

---

## 三、队列与队列集

### 3.1 队列的核心概念

队列是 FreeRTOS 最常用的**任务间通信机制**，你可以把它理解成一个**线程安全的、带阻塞功能的"环形缓冲区"**，作用类似全局变量，但比全局变量安全可控。

先说环形缓冲区。FIFO（先进先出）就像一条**传送带**，数据依次进、依次出，空间循环利用，所以叫环形缓冲区。

- 写指针 `pcWriteTo`：下一个能写入的位置
- 读指针 `pcReadFrom`：下一个能读取的位置
- 指针到末尾就绕回首部，形成"环"

![](/Bzone-blog/images/freertos-source-structure/file-20260530234020675.png)

队列 = FIFO 环形缓冲区 + `Queue_t` 控制块。

**`Queue_t` 里比较重要的字段：**

- **`pcHead` / `pcTail`**：缓冲区的首尾地址
- **`pcWriteTo` / `pcReadFrom`**：读写指针
- **`uxMessagesWaiting`**：当前队列里有多少条消息
- **`uxLength`**：队列最大容量
- **`uxItemSize`**：每条消息多大（字节）
- **`xTasksWaitingToSend` / `xTasksWaitingToReceive`**：等待发送/接收的任务阻塞列表

**拷贝方式：** FreeRTOS 队列是**值拷贝**，发送时把数据复制到缓冲区，接收时再复制出来。好处是发送方和接收方不用共享内存，坏处是大数据拷贝开销大（大数据可以传指针，但内存管理得自己搞）。

### 3.2 队列创建流程

创建队列就两步：

- **`xQueueGenericCreate`**：申请一整块连续内存（`Queue_t` + 缓冲区），缓冲区大小 = `uxLength * uxItemSize`
- **`prvInitialiseNewQueue`**：给控制块赋初值，设好首尾指针、读写指针、消息计数，初始化两个阻塞链表

![](/Bzone-blog/images/freertos-source-structure/file-20260530234437127.png)

`prvInitialiseNewQueue` 就是最后一步，把裸内存变成一个能正常收发、能阻塞唤醒的队列。

![](/Bzone-blog/images/freertos-source-structure/file-20260530235744187.png)

### 3.3 队列的数据填充

#### （1）FIFO 填充（先进先出）

数据从尾部写入、头部读取，最常用的方式。写的时候 `pcWriteTo` 往后移，读的时候 `pcReadFrom` 往后移，到末尾就绕回来。

![](/Bzone-blog/images/freertos-source-structure/file-20260531124503952.png)

#### （2）LIFO 填充（后进先出 / 插入填充）

数据从头部写入，最新的最先被读。适合只关心最新数据的场景（比如传感器采样），用 `xQueueSendToFront()` 就行。

![](/Bzone-blog/images/freertos-source-structure/file-20260531124606868.png)

**（3）信号量/互斥量**：本质也是队列，公用一套函数，只是长度不同，第四章再讲。

![](/Bzone-blog/images/freertos-source-structure/file-20260531124151723.png)

### 3.4 写队列核心流程

`xQueueGenericSend()` 内部分三个阶段：

**（1）快速尝试**：队列有空间吗？有就直接写入，同时看看有没有任务在等接收，有就唤醒。

![](/Bzone-blog/images/freertos-source-structure/file-20260531001040824.png)

**（2）数据拷贝**：把用户数据复制到缓冲区，更新 `pcWriteTo` 和消息计数。

![](/Bzone-blog/images/freertos-source-structure/file-20260531001228878.png)

**（3）唤醒等待者**：看看 `xTasksWaitingToReceive` 链表有没有人等，有的话把优先级最高的唤醒。

![](/Bzone-blog/images/freertos-source-structure/file-20260531001400229.png)

一组图示意整个写队列的过程：

![](/Bzone-blog/images/freertos-source-structure/file-20260531001636418.png)

### 3.5 读队列核心流程

`xQueueReceive()` 和写队列完全对称：

**（1）检查有没有数据**：有就从 `pcReadFrom` 拷贝出来，没有就看 `xTicksToWait` 决定是否阻塞。

![](/Bzone-blog/images/freertos-source-structure/file-20260531001731739.png)

**（2）拷贝完更新指针**：`pcReadFrom` 往后移，消息计数减 1。同时看看有没有任务在等发送，有就唤醒。

![](/Bzone-blog/images/freertos-source-structure/file-20260531001807247.png)

![](/Bzone-blog/images/freertos-source-structure/file-20260531001841033.png)

读队列的拷贝很简单，就是从缓冲区里按 `pcReadFrom` 偏移位置取数据：

![](/Bzone-blog/images/freertos-source-structure/file-20260531124720323.png)

### 3.6 队列集

队列集就是让一个任务**同时等多个队列/信号量**，不用为每个队列开一个任务。

**为什么需要：** 假设一个任务要监听串口、按键、传感器三个队列，没有队列集的话，要么开三个任务，要么在一个任务里轮询（浪费 CPU）。队列集让一个任务同时阻塞在多个队列上，**任何一个有数据就唤醒**，然后再查是哪个。

**用法：**

1. 创建队列集：`xQueueCreateSet(maxItems)`
2. 把队列/信号量加进去：`xQueueAddToSet()`
3. 任务等队列集：`xQueueSelectFromSet()`
4. 查哪个就绪了：`xQueuePeekFromSet()`
5. 从就绪的队列里读数据

**原理：** 队列集本身也是个队列，集合里某个队列有数据写入时，会往队列集里写入一个指向该队列的指针。任务从队列集读到的是就绪队列的句柄，再根据句柄去读实际数据。

> **注意：** V10.0.0 之后官方更推荐用**任务通知**替代队列集，更轻量高效。但在需要同时等多个不同同步原语的场景，队列集还是有用的。

![](/Bzone-blog/images/freertos-source-structure/file-20260531132147737.png)

![](/Bzone-blog/images/freertos-source-structure/file-20260531132217997.png)

### 3.7 队列总结

![](/Bzone-blog/images/freertos-source-structure/file-20260531124027823.png)

---

## 四、信号量

信号量说白了就是**一种特殊的队列**，源码上直接调用队列函数，只不过长度为 1（二值信号量）或由事件计数决定（计数信号量）。但它承担的角色比较特殊，所以单独讲。

> 注意：V10.x 之前用的还是 `xQueueReceive`，V10.x 之后改成了语义更清晰的 `xSemaphoreTake`。

![](/Bzone-blog/images/freertos-source-structure/file-20260531124926391.png)

**信号量的操作就两个：**

- **Take（获取）**：`xSemaphoreTake()`，相当于从队列里读
- **Give（释放）**：`xSemaphoreGive()`，相当于往队列里写

### 4.1 二值信号量

二值信号量就两种状态："有"和"没有"，底层就是一个长度为 1 的队列。

**典型场景：任务同步**

> 任务 A 采集传感器数据，采集完释放信号量；任务 B 等这个信号量，拿到了才开始处理。这样 B 不用轮询，省 CPU。

创建：`xSemaphoreCreateBinary()`

**和互斥量的区别：** 二值信号量**不支持优先级继承**，不能用来保护共享资源。

![](/Bzone-blog/images/freertos-source-structure/file-20260531131629961.png)

### 4.2 计数信号量

计数信号量能记录**多个事件的发生次数**。

**典型场景：**

- **资源池管理**：有 N 个相同的硬件资源，初始值设为 N，每次获取 -1，释放 +1，归零就阻塞
- **事件计数**：中断每次释放信号量，任务获取后就知道发生了多少次事件

创建：`xSemaphoreCreateCounting(uxMaxCount, uxInitialCount)`

![](/Bzone-blog/images/freertos-source-structure/file-20260531131748074.png)

### 4.3 优先级反转问题

这是用信号量时必须知道的经典坑：**高优先级任务可能被低优先级任务间接卡住**。

**怎么发生的：**

1. 低优先级任务 L 拿了互斥量，正在用共享资源
2. 高优先级任务 H 也想拿，被阻塞了（L 还没释放）
3. 中优先级任务 M 就绪了，抢占了 L
4. 结果：H 在等 L，L 被 M 压着，H 的优先级等于被 M "拉低"了

![](/Bzone-blog/images/freertos-source-structure/file-20260531131806412.png)

**解决办法 -- 优先级继承**

H 被阻塞时，L 临时"继承"H 的优先级，尽快跑完释放资源，避免 M 插队。资源释放后 L 恢复原来的优先级。

![](/Bzone-blog/images/freertos-source-structure/file-20260531131914466.png)

### 4.4 互斥量与递归互斥量

互斥量就是带**优先级继承**的二值信号量，专门用来**保护共享资源**。

**互斥量的特殊规则：**

- **谁拿谁放**：获取和释放必须是同一个任务
- **中断里不能用**：中断不能阻塞，互斥量获取可能阻塞

递归互斥量允许同一个任务**多次拿同一把锁**而不会死锁。每次 `TakeRecursive` 内部计数 +1，每次 `GiveRecursive` -1，归零才真正释放。

![](/Bzone-blog/images/freertos-source-structure/file-20260531131958617.png)

![](/Bzone-blog/images/freertos-source-structure/file-20260531132036966.png)

### 4.5 信号量对比总结

| | 二值信号量 | 计数信号量 | 互斥量 | 递归互斥量 |
|---|---|---|---|---|
| 底层 | 长度 1 的队列 | 长度 N 的队列 | 长度 1 + 优先级继承 | 长度 1 + 递归计数 |
| 用途 | 任务同步 | 资源池/事件计数 | 互斥保护 | 递归场景 |
| 优先级继承 | 不支持 | 不支持 | 支持 | 支持 |
| 中断里用 | 可以 | 可以 | 不行 | 不行 |
| 递归获取 | 不行 | 不行 | 不行 | 可以 |
| 谁拿谁放 | 不要求 | 不要求 | 必须 | 必须 |

![](/Bzone-blog/images/freertos-source-structure/file-20260531132131114.png)

---

## 五、事件组

### 5.1 事件组是啥

事件组也是一种同步机制，和信号量不同的是，它玩的是**位操作**。一个事件组变量就是个 `EventBits_t` 整数（通常 32 位），每个位代表一个独立的事件标志。

简单说：**信号量是一个计数器，事件组是一排开关。**

**和信号量的区别：**

| | 信号量 | 事件组 |
|---|---|---|
| 数据模型 | 一个计数器 | 一堆位，每位独立 |
| 一次等什么 | 只能等一个资源 | 能同时等多个事件的组合（AND/OR） |
| 广播通知 | 不行 | 可以（一个位能同时唤醒多个等待任务） |
| 典型场景 | 资源互斥、同步 | 多条件协调（如"等所有传感器就绪才处理"） |

### 5.2 源码结构

**`EventGroup_t` 里的关键字段：**

- **`uxEventBits`**：32 位事件标志，低 24 位存事件状态，高 8 位被内核用了（存待清除的位和等待计数）
- **`xTasksWaitingToSetBits`**：等事件位被置 1 的任务列表
- **`xTasksWaitingToClearBits`**：等事件位被清 0 的任务列表

**核心 API：**

- **`xEventGroupSetBits()`**：把指定位置 1，顺便检查有没有任务在等这些位
- **`xEventGroupClearBits()`**：把指定位清 0
- **`xEventGroupWaitBits()`**：等一个或多个位被置 1（支持 AND/OR）
- **`xEventGroupSync()`**：多任务同步屏障，所有人都到了才一起放行

### 5.3 典型场景

**多传感器数据融合：** 任务要等温度、湿度、气压三个传感器都就绪才处理数据。给三个传感器各分配一个位，任一就绪就 SetBit，任务调 `xEventGroupWaitBits()` 等三个位全为 1。

**多任务同步屏障：** 三个工作线程各算一部分，主线程等它们全算完才汇总。用 `xEventGroupSync()`，每个线程算完就 sync，主线程也 sync，所有人都到了才一起放行。

### 5.4 阻塞机制

事件组的阻塞和队列/信号量不太一样：

- **不需要创建队列**，直接用 `EventGroup_t` 里的变量
- 内核按位匹配，根据 AND/OR 逻辑判断条件是否满足
- 支持超时（`xTicksToWait` 参数）
- 返回时可以选择自动清除已匹配的位（`xClearBitsOnExit`）

---

## 六、任务通知

### 6.1 任务通知是啥

任务通知是 FreeRTOS V8.2.0 引入、V10 版本后极力推荐的**轻量级同步方式**。

**核心概念：** 每个任务的 TCB 里自带一个 32 位的通知值（`ulNotifiedValue`）和一个通知状态（`eNotifyState`）。一个任务发通知，另一个任务收通知，不需要创建队列或信号量。

简单说：**队列/信号量是"快递柜"模式（创建中间人，双方通过它传东西），任务通知是"直接塞到对方口袋"模式（跳过中间人，直接操作对方 TCB 里的字段）。**

### 6.2 任务通知 vs 队列/信号量

| | 队列/信号量 | 任务通知 |
|---|---|---|
| 需要创建控制块吗 | 需要（`Queue_t` 等） | **不需要**，TCB 里自带 |
| 内存开销 | 较大（控制块 + 链表 + 缓冲区） | **几乎没有**（只用 TCB 里的两个字段） |
| 速度 | 需要进临界区操作链表 | **更快**（直接写 TCB 字段） |
| 能传数据吗 | 可以（队列传任意数据） | 可以（传一个 32 位通知值） |
| 能传多个数据吗 | 可以（队列有长度） | **不行**，只有一个值（覆盖式） |
| 能同时等多个源吗 | 可以（队列集） | **不行**，只能等一个任务的通知 |

### 6.3 源码结构

任务通知的实现在 `tasks.c` 里，不涉及任何额外的数据结构：

**发送通知：**

- **`xTaskNotify()`**：直接写目标任务 TCB 里的 `ulNotifiedValue`，并更新 `eNotifyState` 为 `eNotified`
- **`xTaskNotifyGive()`**：等价于 `xTaskNotify(++, eIncrement)`，把通知值 +1，适合做计数型同步
- **`xTaskNotifyFromISR()`**：中断版本，多了个 `pxHigherPriorityTaskWoken` 参数（第八章会讲）

**接收通知：**

- **`xTaskNotifyWait()`**：如果 `eNotifyState == eNotified`，读取通知值并清零状态，返回 `pdTRUE`；否则根据 `ulTicksToWait` 决定是否阻塞
- **`ulTaskNotifyTake()`**：简化版，通知值 > 0 时减 1 并返回，适合计数型场景

### 6.4 典型场景

**替代二值信号量做同步：** 中断里调 `xTaskNotifyGive()`，任务里调 `ulTaskNotifyTake(pdTRUE, portMAX_DELAY)` 等待。和二值信号量效果一样，但不需要创建信号量。

**替代计数信号量做事件计数：** 中断里调 `xTaskNotifyFromISR(0, eIncrement, NULL)`，通知值自动 +1。任务里循环调 `ulTaskNotifyTake(pdTRUE, 0)` 读取并清零，就能知道累计发生了多少次事件。

**传递最新数据（覆盖式）：** 中断里调 `xTaskNotifyFromISR(最新值, eSetValueWithOverwrite, NULL)`，任务里调 `xTaskNotifyWait()` 接收。适合"只关心最新值"的场景，不需要队列。

### 6.5 什么时候该用

**适合：** 一对一通信、不需要队列的缓冲功能、追求最小开销和最快速度、只需要同步或传递一个 32 位值。

**不适合：** 一对多广播（用事件组）、需要缓存多条数据（用队列）、需要同时等多个数据源（用队列集）、需要互斥保护（用互斥量）。

---

## 七、软件定时器

### 7.1 软件定时器是啥

软件定时器就是让你**在指定时间后执行一个回调函数**，不需要硬件定时器资源。

本质上就是个**延迟回调机制**，靠定时器命令队列 + 守护任务实现。

- 创建时指定周期（单次 or 重复）
- 到期后执行你注册的回调函数
- 不占硬件定时器，基于 SysTick 计数实现

### 7.2 源码结构

FreeRTOS 的软件定时器靠三个东西协作：

#### （1）守护任务（Timer Daemon Task）

FreeRTOS 自动创建的内部任务，优先级是 `configTIMER_TASK_PRIORITY`。它的活就是从命令队列里取命令，然后执行（包括到期回调）。

#### （2）定时器命令队列

长度为 `configTIMER_QUEUE_LENGTH` 的队列。所有对定时器的操作（创建、启动、停止等）都是往这个队列发消息，守护任务取出来执行。这样定时器操作就是线程安全的。

#### （3）定时器链表

FreeRTOS 也维护了**两个定时器链表**，和延时链表的双链表原理一样。定时器按到期时间升序排列，最早到期的在链表头部。每次 SysTick 中断时，守护任务检查链表头部是否到期。

### 7.3 回调函数的限制

因为回调是在**守护任务的上下文**里跑的，所以：

- **不能调阻塞 API**：`vTaskDelay()`、`xQueueReceive()` 这些不能用，不然守护任务卡住了，所有定时器都停摆
- **执行时间要短**：别在里面干重活
- 可以 `xSemaphoreGive()`、`xEventGroupSetBits()` 这些非阻塞操作，没问题

### 7.4 两种模式

| | 单次定时器 | 周期定时器 |
|---|---|---|
| 创建参数 | `pdFALSE` | `pdTRUE` |
| 到期后 | 执行回调，自动停 | 执行回调，自动重载，下次到期再执行 |
| 典型场景 | 超时检测、延迟操作 | 周期性采样、LED 闪烁 |

**流程：** 调 `xTimerStart()` → 往命令队列发消息 → 守护任务收到后把定时器插进链表 → SysTick 检查到期没 → 到期就执行回调。

---

## 八、中断管理、临界区与低功耗

### 8.1 FreeRTOS 的中断处理哲学

FreeRTOS 对中断有一个核心原则：**中断里不能做耗时操作，更不能阻塞**。

因为中断会打断所有任务（包括高优先级任务），如果中断里干重活或者阻塞了，整个系统就卡死了。所以 FreeRTOS 的设计思路是：**中断里只做最少的事（标记状态、发通知），然后把真正的处理留给任务去做。**

这就是为什么 FreeRTOS 提供了两套 API：

| | 任务版本 | 中断版本（`FromISR` 后缀） |
|---|---|---|
| 信号量释放 | `xSemaphoreGive()` | `xSemaphoreGiveFromISR()` |
| 队列发送 | `xQueueSend()` | `xQueueSendFromISR()` |
| 任务通知 | `xTaskNotify()` | `xTaskNotifyFromISR()` |
| 能阻塞吗 | 能 | **不能**（`xTicksToWait` 参数被忽略） |

### 8.2 `pxHigherPriorityTaskWoken` 参数

所有 `FromISR` 函数都有一个 `pxHigherPriorityTaskWoken` 参数。它是一个**出参**，用来告诉调用者："我这次操作唤醒了一个更高优先级的任务，你回头得触发一次上下文切换。"

工作流程：

1. 中断里调用 `xSemaphoreGiveFromISR(sem, &xHigherPriorityTaskWoken)`
2. 如果确实唤醒了更高优先级的任务，变量被设为 `pdTRUE`
3. 中断退出前检查：如果为 `pdTRUE`，调用 `portYIELD_FROM_ISR()` 触发 PendSV
4. PendSV 优先级最低，不会阻塞当前中断，等中断处理完才执行切换

**为什么要这么设计？** 中断里不能直接调任务切换函数，切换需要通过触发 PendSV 来延迟执行。这样中断可以尽快退出，切换在中断返回后才发生。

### 8.3 Cortex-M 的中断优先级与嵌套

Cortex-M 内核的中断优先级是**数字越小优先级越高**（和 FreeRTOS 任务优先级相反）。NVIC 硬件自动支持中断嵌套：高优先级中断可以打断低优先级中断。

FreeRTOS 对中断优先级有两个关键配置：

- **`configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY`**：FreeRTOS 能管理的最高优先级（最小数值）。低于这个优先级（数值更大）的中断可以安全调用 `FromISR` API
- **`configKERNEL_INTERRUPT_PRIORITY`**：PendSV 和 SysTick 的优先级，必须设为最低

> **重点理解：** 比如 `configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY = 5`，那么优先级 0-4 的中断不能调任何 FreeRTOS API（因为它们比 FreeRTOS 管理的范围更高），优先级 5-15 的中断可以安全调用 `FromISR` API。

### 8.4 临界区与调度器挂起

多任务系统里，多个任务（或任务和中断）可能同时访问同一个全局变量或数据结构。如果不保护，就会出**数据竞争**。FreeRTOS 提供了两种保护手段：

#### （1）关中断 / 临界区（`taskENTER_CRITICAL`）

操作 Cortex-M 的 **BASEPRI** 寄存器，屏蔽所有优先级等于或低于 `configMAX_SYSCALL_INTERRUPT_PRIORITY` 的中断。

- **不是关全局中断**！更高优先级的中断仍然可以响应
- **关中断的时间要尽量短**（几条到几十条指令）
- 适用场景：修改全局变量、操作链表

```c
taskENTER_CRITICAL();
g_counter++;          // 修改共享变量
vListInsertEnd(...);  // 操作链表
taskEXIT_CRITICAL();
```

#### （2）挂起调度器（`vTaskSuspendAll`）

不关中断，只是把 `uxSchedulerSuspended` 设为非零。调度器在 SysTick/PendSV 中断里检查这个变量，如果被挂起了就跳过任务切换。

- **中断照常响应**，但不会触发任务切换
- **可以嵌套**（内部有计数器）
- 适用场景：较长的临界区、处理大缓冲区、Flash 操作

```c
vTaskSuspendAll();
process_large_buffer(data);  // 耗时操作，中断仍能响应
xTaskResumeAll();            // 恢复调度，切换在这里执行
```

**对比：**

| | 关中断（`taskENTER_CRITICAL`） | 挂起调度器（`vTaskSuspendAll`） |
|---|---|---|
| 中断响应 | **屏蔽**同等及更低优先级中断 | **不屏蔽**，中断照常响应 |
| 任务切换 | 不会切换（中断被屏蔽） | 不会切换（调度器挂起了） |
| 嵌套调用 | 不能在中断里嵌套 | 可以嵌套 |
| 保护时间 | 越短越好（微秒级） | 可以较长（毫秒级） |

> **经验法则：** 几条指令的操作用关中断，耗时操作或需要响应中断的用挂起调度器。

### 8.5 Tickless 低功耗模式

嵌入式设备很多是电池供电的，功耗是关键指标。

默认情况下，SysTick 中断**每 tick（通常 1ms）触发一次**，每次都唤醒 MCU 检查任务状态。如果有任务延时 10 秒，中间 SysTick 白白唤醒了 10000 次。

**Tickless 的核心思想：** 当系统进入空闲任务时，**停止 SysTick 中断**，让 MCU 进入深度休眠，直到下一个任务到期或有外部中断才唤醒。空闲期间几乎零功耗。

**工作流程：**

1. 空闲任务调用 `portSUPPRESS_TICKS_AND_SLEEP(ExpectedIdleTime)`
2. 计算要暂停多少个 tick，配置 SysTick 在到期时唤醒 MCU
3. 调用 `__wfi()` 或 `__wfe()` 指令，MCU 进入低功耗休眠
4. MCU 被唤醒（SysTick 到期或外部中断）
5. **补偿 `xTickCount`**：直接把 `xTickCount` 加上睡了多久（不是一个个 tick 累加），所有延时链表上的任务到期时间自动对齐

**配置选项（`FreeRTOSConfig.h`）：**

- **`configUSE_TICKLESS_IDLE`**：设为 1 启用
- **`configEXPECTED_IDLE_TIME_BEFORE_SLEEP`**：最少空闲几个 tick 才进入睡眠（太短不值得睡）
- **`configPRE_SLEEP_PROCESSING(x)`** / **`configPOST_SLEEP_PROCESSING(x)`**：进入/退出睡眠前的回调，可以关闭外设时钟进一步省电

**注意事项：**

- 只能在空闲任务里触发（所有任务都阻塞了才跑空闲任务）
- MCU 从深度休眠唤醒需要时间（几微秒到几毫秒），这段时间 SysTick 不准
- 具体能用哪种休眠模式（Sleep / Stop / Standby）取决于硬件
- 调试时建议关闭，Tickless 会让调试器难以跟踪

---

## 九、内存管理

### 9.1 为什么要知道这个

FreeRTOS 动态创建任务、队列、信号量都要调 `pvPortMalloc()`，释放调 `vPortFree()`。FreeRTOS 提供了 **5 种堆管理方案**（heap_1 到 heap_5），选哪个取决于你的需求：要不要释放内存？担不担心碎片？内存大小固定不？

### 9.2 五种方案一览

| 方案 | 能分配 | 能释放 | 碎片 | 适用场景 |
|---|---|---|---|---|
| **heap_1** | 能 | 不能 | 无碎片 | 内存只分配不释放 |
| **heap_2** | 能 | 能 | 有碎片 | 分配/释放的块大小基本固定 |
| **heap_3** | 能 | 能 | 看底层 | 封装标准库 `malloc`/`free` |
| **heap_4** | 能 | 能 | 低碎片 | **最常用**，支持空闲块合并 |
| **heap_5** | 能 | 能 | 低碎片 | 支持**非连续内存区域** |

我画了一张图来帮助大家理解堆栈的概念：
![](/Bzone-blog/images/freertos-source-structure/file-20260601215343130.png)
### 9.3 heap_1：最简单

只实现 `pvPortMalloc()`，**不支持释放**。维护一个分配指针，每次分配就往前走。

```c
// heap_1 的核心逻辑（简化）
static uint8_t ucHeap[configTOTAL_HEAP_SIZE];
static size_t xNextFreeByte = 0;

void *pvPortMalloc(size_t xWantedSize) {
    xWantedSize = (xWantedSize + 7) & ~7;  // 对齐到 8 字节
    if (xNextFreeByte + xWantedSize <= configTOTAL_HEAP_SIZE) {
        void *p = &ucHeap[xNextFreeByte];
        xNextFreeByte += xWantedSize;
        return p;
    }
    return NULL;
}
```

**优点：** 零碎片，分配极快（O(1)），不需要加锁。
**适合：** 任务启动时全部创建好，运行期间不再创建/删除。很多安全关键系统（航空、汽车电子）用的就是这个。

### 9.4 heap_2：最佳适配

支持分配和释放，维护一个**空闲块链表**。分配时遍历找**大小最接近**的块（最佳适配），如果块太大就分割，剩余的放回空闲链表。

**问题：** 不做空闲块合并。释放的内存和相邻空闲块无法合成一个大块，会产生**外部碎片**。

**适合：** 分配/释放的大小基本固定（比如总分配相同大小的 TCB）。

### 9.5 heap_3：标准库封装

直接包一层 C 标准库的 `malloc()`/`free()`，加上 `vTaskSuspendAll()` 保护。

```c
void *pvPortMalloc(size_t xWantedSize) {
    void *pvReturn;
    vTaskSuspendAll();
    pvReturn = malloc(xWantedSize);
    xTaskResumeAll();
    return pvReturn;
}
```

**优点：** 简单，利用标准库成熟的碎片管理。
**缺点：** 标准库 `malloc` 需要额外堆开销（几十到几百字节），内存紧张的系统不适用。

### 9.6 heap_4：最常用

在 heap_2 基础上加了**相邻空闲块合并**。释放时检查前后空闲块是否相邻，相邻就合并成一个大块。碎片率远低于 heap_2。

分配用**首次适配**（找到第一个够大的块就用），速度比 heap_2 的最佳适配更快。

```c
// heap_4 的释放逻辑（简化）
void vPortFree(void *pv) {
    // 找到这个块在空闲链表中的位置
    // 检查前一个空闲块是否相邻 → 合并
    // 检查后一个空闲块是否相邻 → 合并
    // 插入空闲链表
}
```

**大多数 FreeRTOS 项目的默认选择。**

### 9.7 heap_5：非连续内存

heap_4 只能管一块连续内存，heap_5 允许把**多块不连续的内存**统一管理（比如 STM32 内部 SRAM + 外部 SDRAM 混着用）。

```c
const HeapRegion_t xHeapRegions[] = {
    { (uint8_t *)0x20000000, 0x10000 },  // 内部 SRAM 64KB
    { (uint8_t *)0x60000000, 0x10000 },  // 外部 SDRAM 64KB
    { NULL, 0 }
};
vPortDefineHeapRegions(xHeapRegions);
```

heap_5 内部会把这几块内存串联成统一的空闲链表，之后用法和 heap_4 一样。

### 9.8 怎么选

```
要释放内存吗？
├── 否 → heap_1（零碎片，最高效）
└── 是 → 内存不连续？
    ├── 是 → heap_5
    └── 否 → 分配/释放大小基本固定？
        ├── 是 → heap_2
        └── 否 → heap_4（推荐）
```

heap_3 一般不推荐，除非你特别依赖标准库的 `malloc` 行为。

> **建议：** 不管选哪个，开发阶段都打开栈溢出检测（`configCHECK_FOR_STACK_OVERFLOW`）和 `vApplicationMallocFailedHook()`，内存不够了能及时发现。

---

---

到这里，FreeRTOS 的源码结构就讲完了。用一段话把核心模块串起来：

**底层基础：** 链表（`List_t`）是整个 FreeRTOS 的骨架，就绪列表、阻塞列表、挂起列表、定时器链表全靠它组织。

**任务管理：** 创建任务就是填充 TCB + 插入就绪列表。调度器从就绪列表里挑最高优先级的任务跑。任务通过 PendSV 中断切换，入栈/出栈保存恢复寄存器上下文。

**通信与同步：**
- **队列**是最基础的通信机制（值拷贝的环形缓冲区），信号量和互斥量本质上都是特殊的队列
- **事件组**用位操作实现多条件同步，适合"等多个事件都就绪"的场景
- **任务通知**是最轻量的同步方式（直接写对方 TCB），一对一场景优先考虑

**定时与功耗：** 软件定时器靠"命令队列 + 守护任务"实现延迟回调；Tickless 模式在空闲时停 SysTick、进深度休眠省电。

**系统防护：** 中断里只做最少的事（FromISR API），通过 `pxHigherPriorityTaskWoken` 触发延迟切换。临界区用关中断（短操作）或挂起调度器（长操作）保护共享资源。

**内存管理：** heap_1 到 heap_5 五种方案，按"要不要释放"和"内存是否连续"选型。

> **一句话总结：FreeRTOS 的源码结构就是"用链表管理一切"** -- 任务是链表节点，队列是环形缓冲区，信号量是特殊队列，定时器是延迟回调，调度器是链表遍历。理解了链表和 TCB，就理解了 FreeRTOS 的半壁江山。

![](/Bzone-blog/images/freertos-source-structure/file-20260530191616821.png)
