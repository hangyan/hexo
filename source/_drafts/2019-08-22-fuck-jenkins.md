---
title: 失败博物馆 - Jenkins
toc: true
categories: 技术
date: 2019-08-22 17:02:37
excerpt: ...
tags:
---

我想记录一下工作中使用过的软件里的一些在设计上比较失败的。因为它们，工作中多了很多痛苦。

Jenkins 便是其中之一。跟很多软件一样，Jenkins 在商业上是成功的，成功的原因就是用户并没有什么选择。对 Jenkins 非常了解的人来说，能列举出非常多的详细的jenkins 的问题。从一个普通使用者的角度来看，最直观的两个感觉就是: Jenkinsfile 难写，流水线很慢。

对我来讲，Jenkinsfile 就是最大的设计败笔。从工作中的反馈来看，即使看了文档，也没人知道怎么去写一个正确的 Jenkinsfile。用户想描述好一些流水线步骤，最直观的想法应该就是贴近于 Makefile, 每个步骤几条命令，清清楚楚。再加上其他一些并发控制，产出物管理即可。用 YAML 或者 TOML 这样的配置文件即可。



几乎除了 Jenkins 之外的所有 CI/CD 工具，都选择了 用YAML。而 Jenkins 使用了 Jenkinsfile, Groovy 语言，意味着用户需要首先了解 Groovy，在学习 jenkins的语法，才能写好 Jenkinsfile。



