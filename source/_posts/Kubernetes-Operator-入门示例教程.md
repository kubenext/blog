---
title: Kubernetes Operator 入门示例教程
date: 2019-09-20 22:57:32
top: true 
categories: Kubernetes
tags: 
- Kubernetes
- Operator
---

为了更容易构建Kubernetes应用程序，Red Hat和Kubernetes开源社区发布了[Operator Framework](https://github.com/operator-framework)，并提供了应用市场[operatorhub](https://operatorhub.io)。旨在以更有效，自动化和可扩展的方式管理Kubernetes应用程序，称为`Operator`。

> 在本篇文章中，我们将进行简单示例来演示如何使用`Operator Framework`框架来开发一个Operator应用。

## Operators & Kubernetes applications

Operator 是一种打包，部署和管理Kubernetes应用程序的开源工具包。以前我们在Kubernetes集群上面部署应用程序，往往需要通过Kubernetes API和kubectl工具进行管理、部署。一个应用程序要运行在Kubernetes上面，大部分情况下，我们需要编写deployment、service以及ingree等API才能正确运行,虽然它复杂度不高，但大家可以想想如etcd,mysql,redis等可生产的集群部署，我们需要考虑非常多的因素才能正确运行。虽然以上这些部署我们都能顺利完成，但它却没有可移植性，不能多集群复制，不能分发共享给其它组织运行。

Operator 将上面的各种配置将其编码为更容易打包、分发与运行的软件。它比运维人员的人工判断要敏捷的多，它可以观测集群/应用的当前状态并在若干毫秒之内作出合理的运维决定。Operator遵循如下成熟度模型：

![](/images/0001.webp)