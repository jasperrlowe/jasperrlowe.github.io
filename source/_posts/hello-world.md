---
title: 第一次注册：全人助手已经在路上
date: 2025-10-02 10:00:00
slug: hello-world
categories:
  - Diary
tags:
  - CUHK-SZ
  - Holistic Assistant
---

这天是我第一次把博客搭起来、第一次把内容部署出去。

但其实在这之前，`全人助手（Holistic Assistant）` 已经在推进了：我想做一个面向 CUHK-SZ 学生的 AI 学术顾问系统，把"培养方案 PDF + SIS"里分散的课程信息变成可用的知识库，再接进 AI 问答与学习路线图。

<!-- more -->

目标是让它不只是个"搜索引擎"，而是能真正理解 CUHK-SZ 的课程体系、能给出贴合实际的学习建议、能帮学生在选课前就做好决策。

这篇博客就当作起点：之后我会把项目从 **v0** 到 **可用产品** 的过程，按里程碑持续记录下来。

---

## 📚 技术准备

在启动这个项目前的几周，我开始系统性学习构建这个系统所需的技术栈：

- **Flask**：从官方 Quickstart 开始，学习 RESTful API 设计
- **Gemini API**：阅读 Google AI 文档，了解如何调用生成式 AI
- **PDF 解析**：研究 PyPDF2 和文本提取的常见问题
- **前端交互**：学习 Tailwind CSS 和现代 JS（ES6+）

---

## 🙏 致谢

- **严明教授**和**贾建民教授**：在项目早期给予了宝贵的建议和支持
- **团队成员**：感谢几位同组成员在初期帮助采集培养方案 PDF 等基础数据

---

## Quick Start（给未来的自己）

### 创建新文章

``` bash
$ hexo new "My New Post"
```

### 本地预览

``` bash
$ hexo server
```

### 生成静态文件

``` bash
$ hexo generate
```

### 部署到远程

``` bash
$ hexo deploy
```
