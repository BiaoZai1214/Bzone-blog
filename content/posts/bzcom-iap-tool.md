---
title: "标仔OTA调试助手"
date: 2026-02-15
draft: false
description: "Qt开发的串口+TCP+CAN多模OTA调试助手"
tags: ["Qt", "OTA", "IAP", "串口", "TCP", "CAN", "Bootloader"]
cover: "/Bzone-blog/images/BzCOM-cover.png"
categories: ["上位机"]
---

## 项目概况

这是一个基于Qt开发，支持 **串口 + TCP + CAN** 的多功能 IAP/OTA 上位机工具，吸收了市面上串口调试工具的优点，主打简洁舒适的使用体验~

![界面预览](/Bzone-blog/images/file-20260615201211296.png)

文件包中，有用野火F103小智做的Bootloader例程项目，下位机部分的教程，可以看尚硅谷的课程~

> **特别说明：**
> 通常的项目串口发送用的是单缓冲，但是因为F103的Flash只有1个Bank，写入数据的时候占用CPU，无法进入ISR，所以项目用了双缓冲的设计

## 项目链接

- GitHub：https://github.com/BiaoZai1214/BzCOM
- 百度云：https://pan.baidu.com/s/143V7Ot-IjoMvE1truXqNJQ?pwd=cz58

> 本项目对所有非商业的个人和企业完全开放，可以进行修改和教学参考，但希望能够保留冠名 -- **标仔**。

## 功能介绍

Tab标签支持3模切换，对串口和TCP做了OTA支持，CAN的总线结构不太适合做OTA，所以仅保留了通讯功能。外接一个PCAN的USB转CAN模块即可实现PC与单片机的CAN通讯。

![功能界面](/Bzone-blog/images/file-20260616113933259.png)

## 升级流程

OTA走的是CRC16校验；IAP走的是无协议，STM32端按下KEY1发送数字选择 IAP / OTA 模式，按下KEY2直接进入IAP模式。

![升级流程图](/Bzone-blog/images/file-20260616114730651.png)

## TCP通信

![TCP通信](/Bzone-blog/images/5a136581fff9a179e537033193674d62.jpg)
