---
title: 从一个真实的案例看snakeyaml反序列化漏洞
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

## snakeyaml简介

snakeyaml 是 java 中常用来处理 yaml 格式文件的库。

{% codeblock %}
alert('Hello World!');
{% endcodeblock %}

