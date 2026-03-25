---
title: "Hugo 博客搭建指南"
date: 2026-03-23
draft: false
tags: ["Hugo", "静态网站", "教程"]
categories: ["技术"]
author: "Bz"
description: "详细讲解如何使用 Hugo 快速搭建个人博客，包括主题安装和 GitHub Pages 部署"
coverColor: "#11998e"
coverEmoji: "📝"
showReadingTime: true
featureimage: "https://images.unsplash.com/photo-1499750310107-5fef28a66643?w=600&q=80"
---

本文介绍如何使用 Hugo 快速搭建个人博客。

## 安装 Hugo

访问 [Hugo Releases](https://github.com/gohugoio/hugo/releases) 下载对应系统的二进制文件。

```bash
# 验证安装
hugo version
```

## 创建新站点

```bash
hugo new site my-blog
cd my-blog
```

## 启动本地服务器

```bash
hugo server -D
```

访问 `http://localhost:1313` 即可预览。
