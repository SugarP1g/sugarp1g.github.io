---
title: 【未完成】从一个真实的案例看snakeyaml反序列化漏洞
comments: true
tags:
  - Java
  - Yaml
  - 反序列化
categories: 漏洞分析
date: 2022-01-16 20:29:43
---

最近碰到一个产品在使用snakeyaml时，使用黑名单的方式对yaml进行校验。这篇文章记录下绕过黑名单实现RCE的整个过程。
<!-- more -->

## snakeyaml漏洞简介

snakeyaml反序列化漏洞的基本原理就不再赘述了，网上有很多的分析文章。如果不了解的，可以看下这篇文章：[Java SnakeYaml反序列化漏洞](https://www.mi1k7ea.com/2019/11/29/Java-SnakeYaml%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/)。
