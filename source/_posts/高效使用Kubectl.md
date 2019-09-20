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

在学习如何更高效地使用 kubectl 之前，我们应该去了解下 kubectl 是什么已经它是如何工作的。从用户角度来看，kubectl是控制Kubernetes的客户端。它可以执行Kubernetes的所有操作。从用户角度来说，kubectl 就是控制 Kubernetes 的驾驶舱，它允许你执行所有可能的 Kubernetes 操作；从技术角度来看，kubectl 就是 Kubernetes API 的一个客户端而已。

Kubernetes API 是一个 HTTP REST API 服务，该 API 服务才是 Kubernetes 的真正的用户接口，Kubernetes 通过该 API 进行实际的控制。这也就意味着每个 Kubernetes 的操作都会通过 API 端点暴露出去，当然也就可以通过对这些 API 端口进行 HTTP 请求来执行相应的操作。

所以，kubectl 最主要的工作就是执行 Kubernetes API 的 HTTP 请求：

![](/images/0002.svg)

Kubernetes 是一个完全以资源为中的系统，Kubernetes 维护资源的内部状态，所有 Kubernetes 的操作都是对这些资源的 [CRUD](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/) 操作，你可以通过操作这些资源来完全控制 Kubernetes（Kubernetes 会根据当前的资源状态来确定需要做什么）。

比如下面的例子。

假如现在我们想要创建一个 ReplicaSet 的资源，然后创建一个名为 replicaset.yaml 的资源文件来定义 ReplicaSet，然后运行下面的命令：

```
kubectl create -f replicaset.yaml
```

这个命令执行后会在 Kubernetes 中创建一个 ReplicaSet 的资源，但是这幕后发生了什么呢？

Kubernetes 具有创建 ReplicaSet 的操作，并且和其他 Kubernetes 操作一样的，也是通过 API 端点暴露出来的，上面的操作会通过指定的 API 端口进行如下的操作：

```
POST /apis/apps/v1/namespaces/{namespace}/replicasets
```

我们可以在 API 文档中找到所有 Kubernetes 操作的 API Endpoint（包括上面的端点），要向 Endpoint 发起实际的请求，我们需要将 APIServer 的 URL 添加到 API 文档中列出的 Endpoint 路径中。

所以，当我们执行上面的命令时，kubectl 会向上面的 API Endpoint 发起一个 HTTP POST 请求，ReplicaSet 的定义（replicaset.yaml 文件中的内容）通过请求的 body 进行传递。这就是 kubectl 与 Kubernetes 集群交互的命令如何工作的。

> 我们也完全可以使用 curl 等工具手动的向 Kubernetes API 发起 HTTP 请求，kubectl 只是让我们更加容易地使用 Kubernetes API 了。

这些是 kubectl 的最基础的知识点，但是每个 kubectl 操作者还有很多 Kubernetes API 的知识点需要了解，所以我们这里再简要介绍一下 Kubernetes 的内部结构。

## Kubernetes 架构

Kubernetes 由一组独立的组件组成，这些组件在集群的节点上作为单独的进行运行，有些组件在 Master 节点上运行，有一些组件在 Node 节点上运行，每个组件都有一些特定的功能。

Master 节点上最主要的组件有下面几个：

- etcd: 存储后端，整个集群的资源信息都存在 etcd 里面
- kube-apiserver: 提供给整个集群的 API 服务，是唯一一个直接和 etcd 进行交互的组件
- kube-controller-manager: 控制器，主要是确保资源状态符合期望值
- kube-scheduler: 调度器，将 Pod 调度到工作节点

Node 节点上最重要的组件：

- kubelet: 管理工作节点上的容器

为了了解这些组件之间是如何协同工作的，我们再来看下上面的例子，假如我们执行了上面的kubectl create -f replicaset.yaml命令，kubectl 对创建 ReplicaSet 的 API Endpoint 发起了一个 HTTP POST 请求，这个时候我们的集群有什么变化呢？看下面的演示：

![](/images/0003.svg)

> 在执行 kubectl create -f replicaset.yaml 命令之后，APIServer 将 ReplicaSet 的资源清单保存在了 etcd 中。

