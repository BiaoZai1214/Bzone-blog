---
title: "FreeRTOS 任务优先级与调度"
date: 2026-03-21
draft: false
tags: ["FreeRTOS", "RTOS", "嵌入式"]
categories: ["技术"]
author: "Bz"
description: "解析 FreeRTOS 任务调度的核心原理，优先级反转与抢断式调度"
coverColor: "#0c3483"
coverEmoji: "⚙️"
showReadingTime: true
featureimage: "https://images.unsplash.com/photo-1518770660439-4636190af475?w=600&q=80"
---

FreeRTOS 是最流行的开源实时操作系统。

## 任务状态机

```
就绪 → 运行 → 阻塞
  ↑______|      ↓
         挂起 ←┘
```

## 创建任务

```c
TaskHandle_t xTaskHandle;
xTaskCreate(
    vTaskFunction,      // 任务函数
    "Task1",            // 任务名称
    1000,               // 栈深度
    NULL,               // 参数
    1,                  // 优先级
    &xTaskHandle        // 句柄
);
```

## 调度策略

- **抢断式** - 高优先级任务可以打断低优先级
- **时间片轮转** - 同优先级任务轮流执行
- **优先级继承** - 解决优先级反转问题

合理设置任务优先级是 RTOS 开发的关键！
