---
title: "FreeRTOS 任务调度：优先级与状态机解析"
date: 2026-03-23
draft: false
description: "深入解析 FreeRTOS 任务调度的核心原理，包括任务状态机、优先级配置和抢占式调度机制。"
tags: ["FreeRTOS", "RTOS", "嵌入式"]
categories: ["技术"]
author: "Bz"
coverColor: "#0c3483"
coverEmoji: "🚀"
showReadingTime: true
featureimage: "https://images.unsplash.com/photo-1518770660439-4636190af475?w=600&q=80"
---

FreeRTOS 是最流行的开源实时操作系统，广泛应用于 MCU 项目中。本文深入解析其任务调度机制。

## 任务状态机

```
       ┌──────┐
       │ 运行 │
       └──┬───┘
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
┌──────┐┌──────┐┌──────┐
│ 就绪 ││ 阻塞 ││ 挂起 │
└──────┘└──────┘└──────┘
```

## 创建任务

```c
#include "FreeRTOS.h"
#include "task.h"

void vTaskFunction(void *pvParameters) {
    while (1) {
        // 任务逻辑
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

// 创建任务
TaskHandle_t xTaskHandle = NULL;
xTaskCreate(
    vTaskFunction,      // 任务函数
    "Task1",            // 任务名称
    1000,               // 栈深度(字)
    NULL,               // 参数
    1,                  // 优先级 (0-configMAX_PRIORITIES-1)
    &xTaskHandle        // 句柄
);
```

## 调度策略

- **抢占式调度** - 高优先级任务可打断低优先级任务
- **时间片轮转** - 同优先级任务轮流执行
- **优先级继承** - 解决优先级反转问题

## 注意事项

1. 优先级数值越大优先级越高
2. 任务栈空间要合理估算
3. 避免在中断中调用阻塞 API

掌握任务调度是 RTOS 开发的核心技能！
