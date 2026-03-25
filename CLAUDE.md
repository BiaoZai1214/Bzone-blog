# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

基于 Hugo + Blowfish 主题的中文个人博客，主题作为 git submodule 管理。Hugo 版本需 >= 0.141.0。

## 常用命令

```bash
# 本地开发预览
hugo server

# 生产环境构建
hugo

# 创建新文章
hugo new posts/<文件名>.md

# 更新主题子模块
git submodule update --remote themes/blowfish
```

## 配置架构

配置分为两个层面：
- **hugo.toml**: 站点级配置（语言、作者、分页、社交链接），其中 `[params]` 为兼容性别名
- **config/_default/**: Blowfish 主题配置（优先级更高）
  - `params.toml`: 主题参数（色彩方案、布局、首页卡片、页脚等）
  - `menus.zh.toml`: 导航菜单（覆盖 hugo.toml 中的菜单定义）

注意：主题参数在两处都有定义时，`config/_default/params.toml` 优先生效。

## 文章结构

文章放在 `content/posts/` 目录，front matter 常用字段：

```yaml
---
title: "文章标题"
date: 2026-03-21
draft: false
tags: ["FreeRTOS", "嵌入式"]
categories: ["技术"]
author: "Bz"
description: "文章描述（SEO 用）"
coverColor: "#0c3483"      # 卡片背景色
coverEmoji: "⚙️"           # 封面 emoji 图标
featureimage: "https://..." # 封面图片 URL
showReadingTime: true
---
```

## 主题覆盖机制

Hugo 的 `layouts/` 目录会覆盖主题同名文件：
- `layouts/_default/list.html` — 自定义文章列表/标签页渲染逻辑
- `layouts/partials/extend-head.html` — 注入自定义 CSS 链接
- `layouts/partials/home/card.html` — 自定义首页卡片
- `layouts/partials/article-link/card.html` — 自定义文章卡片

自定义 CSS 文件放在 `assets/css/extended.css`，通过 `extend-head.html` 引用。

## 主题文档

https://blowfish.page/zh-cn/docs/
