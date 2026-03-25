---
title: "I2C 协议从零理解"
date: 2026-03-18
draft: false
tags: ["I2C", "通信协议", "嵌入式"]
categories: ["技术"]
author: "Bz"
description: "详解 I2C 的起始停止条件、地址格式和常见错误排查"
coverColor: "#2196F3"
coverEmoji: "🔗"
showReadingTime: true
featureimage: "https://images.unsplash.com/photo-1518770660439-4636190af475?w=600&q=80"
---

I2C 是最常用的板级通信协议，只需要两根线。

## 信号线

- **SCL** - 时钟线
- **SDA** - 数据线

## 传输格式

```
S + 从机地址(7位) + R/W(1位) + ACK + 数据 + ACK + P
```

## 关键时序

1. **起始条件** - SCL 高时 SDA 拉低
2. **停止条件** - SCL 高时 SDA 拉高
3. **应答位** - 每字节传输后接收方拉低 SDA

## 常见问题

- 总线被拉死？检查上拉电阻是否连接
- 通信不稳？降低 I2C 时钟频率
- 从机无响应？检查地址是否正确
