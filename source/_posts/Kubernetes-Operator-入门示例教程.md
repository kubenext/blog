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

> Helm通常用于Charts的部署于升级，Ansible则可以触及到应用的生命周期管理，而高级的Operator可以实现无缝升级、自动处理故障，真正达到Auto-pilot，自运维、自巡航。

## 开始我们的示例

> 该示例将目标是完成部署一个deployment、service等，完成运维人员部署一个无状态应用的过程。

### 环境搭建

- Kubernetes 集群
- Golang
- operator-sdk
- dep

```bash
brew install docker
brew install go
# 可选，以下我们采用GO Modules管理依赖s
brew install dep
brew install operator-sdk
```

管理operator-sdk其它安装方法，请参考官方文档。[](https://github.com/operator-framework/operator-sdk/blob/master/doc/user/install-operator-sdk.md)

### 创建项目

环境安装好后，我们就可以使用opertator-sdk创建一个新项目了。

```bash
operator-sdk new lear-operator --repo github.com/kubenext/lear-operator --git-init
cd lear-operator
tree

.
├── build
│   ├── Dockerfile
│   └── bin
├── cmd # main.go,项目入口目录
│   └── manager
├── deploy # Kubernetes资源清单
│   ├── operator.yaml
│   ├── role.yaml
│   ├── role_binding.yaml
│   └── service_account.yaml
├── go.mod
├── go.sum
├── pkg # 主要编码目录
│   ├── apis # 自定义的API和CRD等
│   └── controller # 控制器逻辑代码就在这个目录
├── tools.go
└── version
    └── version.go

9 directories, 9 files
```

以上项目采用go语言的modules管理项目， 所以必须输入`--repo`参数，`--git-init`初始化git仓库。


### 创建API

当项目创建好的时候，我们就可以通过sdk来增加一个自定义的CRD资源了。

```bash
operator-sdk add api --api-version=lear.example.com/v1alpha1 --LearService
```
> 通过以上命令，就是创建了CRD对应的资源结构框架，后面我们需要定义需要的结构体。

### 创建Controller控制器

```bash
operator-sdk add controller --api-version=lear.example.com/v1alpha1 --kind=LearService
```

整个项目基础框架已经搭建完成了，我们有一个CRD API，并且也有了控制器。接下来，我们就开始定义并编码我们的控制器逻辑了。


