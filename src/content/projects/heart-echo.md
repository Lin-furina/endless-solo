---
title: "HeartEcho · AI 情感陪伴"
slug: heart-echo
published: 2026-10-30
order: 1
description: "三人团队四周完成的 AI 情感陪伴 Web 应用。核心是做一个「像真人一样」的 AI 伴侣——有记忆、有情绪、会主动找你聊天。我负责前端全链路。"
image: ""
status: "published"
tags:
  - Vue3
  - TypeScript
  - Go
  - Pinia
link:
  - label: "源码"
    icon: "fa7-brands:github"
    value: "https://github.com/nanirise/heart-echo"
---

## 项目简介

HeartEcho（心回响）是一个 AI 情感陪伴 Web 应用。用户可以创建自定义人设的 AI 伴侣，它能记住对话内容、维持情绪线索，并在合适的时候主动发起交流。

## 技术栈

- **前端**：Vue 3 + TypeScript + Pinia + Vue Router + Element Plus + Vite
- **后端**：Go + Gin + GORM
- **AI 服务**：Python
- **部署**：Docker Compose + Nginx

## 我负责的部分

前端全链路：类型契约层、请求层与登录态、路由与布局，以及后续的聊天、记忆、朋友圈页面。

这几个模块的设计细节写在文章里：[HeartEcho 前端：先把契约、请求和登录态搭稳](/posts/heart-echo-frontend/)
