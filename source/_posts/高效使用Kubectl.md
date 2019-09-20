---
title: 高效使用Kubectl
date: 2019-09-21 02:43:41
top: true 
categories: Kubernetes
tags: 
- Kubernetes
- Kubectl
---

每当你话费大量时间使用特定工具时，都应该了解它并学习如何有效地使用它。
本文包含一些列提高和技巧，使你对kubectl的使用更加高效。同时，它旨在加深你对Kubernetes各个工作细节的理解。
本文的目标不仅使你使用Kubernetes的日常工作更加高效，而且更加愉快！

## 什么是kubectl？

在学习如何更有效地使用kubectl之前，你应该了解它是什么以及如何工作的。

从用户角度来看，kubectl是控制Kubernetes的客户端。它可以执行Kubernetes的所有操作。

从技术角度来看，kubectl是Kubernetes API的客户端。

Kubernetes API是HTTP REST API。因此kubectl的就是对Kubernetes API的请求操作。

![](/images/0002.svg)
