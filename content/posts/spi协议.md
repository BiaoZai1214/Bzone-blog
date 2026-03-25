---
title: "SPI 协议深度剖析"
date: 2026-03-19
draft: false
tags: ["SPI", "通信协议", "嵌入式"]
categories: ["技术"]
author: "Bz"
description: "详解 SPI 四种模式、时钟极性和相位配置，适合新手入门"
coverColor: "#30cfd0"
coverEmoji: "💾"
showReadingTime: true
featureimage: "https://images.unsplash.com/photo-1518770660439-4636190af475?w=600&q=80"
---

SPI 是嵌入式最常用的高速通信协议之一。

## 硬件连接

| 引脚 | 说明 |
|------|------|
| MOSI | 主出从入 |
| MISO | 主入从出 |
| SCK | 时钟信号 |
| CS/SS | 片选信号 |

## 四种模式

| 模式 | CPOL | CPHA |
|------|------|------|
| 0 | 0 | 0 |
| 1 | 0 | 1 |
| 2 | 1 | 0 |
| 3 | 1 | 1 |

## 驱动开发要点

1. 配置正确的时钟极性和相位
2. 片选信号要手动控制还是硬件自动
3. 考虑 DMA 提升传输效率
