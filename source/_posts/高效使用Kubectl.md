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

![](/images/0004.svg)

> 这将触发控制器管理器中的ReplicaSet控制器，该控制器监视ReplicaSet资源的创建，更新和删除。

![](/images/0005.svg)

> ReplicaSet控制器为ReplicaSet的每个副本创建Pod定义（根据ReplicaSet定义中的Pod模板），并将其保存在存储后端中。

![](/images/0006.svg)

> 这将触发调度程序，该调度程序监视尚未分配给工作程序节点的Pod。

![](/images/0007.svg)

> 调度程序为每个Pod选择一个合适的工作节点，并将此信息添加到存储后端的Pod定义中。

![](/images/0008.svg)

> 这将触发Pod已调度到的工作程序节点上的kubelet，后者监视已调度到其工作程序节点的Pod。

![](/images/0009.svg)

> kubelet从存储后端读取Pod定义，并指示容器运行时（例如Docker）在工作节点上运行容器。

经过上面的几个步骤 ReplicaSet 就可以运行起来了。

## Kubernetes API的作用

从上面的示例中我们可以看出，Kubernetes 组件（APIServer 和 etcd 除外）都是通过监听后端的资源变化来工作的。但是这些组件是不会直接访问 etcd 的，只能通过 APIServer 去访问 etcd。我们再回顾下上面的资源创建过程：

- ReplicaSet 控制器使用 ReplicaSets API 的 List 操作和 watch 参数来监听 ReplicaSet 资源的变化。
- ReplicaSet 控制器使用 create Pod API 来创建 Pod。
- 调度器通过 patch Pod API 来更新 Pod 和工作节点相关的信息。

我们使用 kubectl 来操作也是使用这些相同的 API 的。有了这些知识点后，我们就可以总结出 Kubernetes 的工作原理了：

- 存储后端（etcd）存储 Kubernetes 的资源。
- APIServer 通过 Kubernetes API 的形式提供存储后端的相关接口。
- 所有其他组件和外部用户都是通过 Kubernetes API 来读取、监听和操作 Kubernetes 的资源。

> 熟悉这些基本概念将会有助于我们更好地理解 kubectl。

## 命令提示、补全

命令提示（补全）是提高生产力最有用但也是经常被忽略的技巧之一。命令补全允许你使用 tab 键自动补全 kubectl 的相关命令，包括子命令、选项和参数，以及资源名称等一些复杂的内容。

在下图中我们可以看到使用 kubectl 命令自动补全的相关演示：

![](/images/0010.gif)

命令补全可用于 Bash 和 Zsh shell 终端。

官方文档中就包含了一些关于命令补全的相关说明，当然也可以参考下面我们为你提供的一些内容。

## 命令补全的工作原理

一般来说，命令补全是通过执行一个补全脚本的 shell 功能，补全脚本也是一个 shell 脚本，用于定义特定命令的补全功能。

kubectl 在 Bash 和 Zsh 下可以使用下面的命令自动生成并打印出补全脚本：

```bash
kubectl completion bash
kubectl completion zsh
```

理论上在合适的 shell 中 source 上面命令的输出就可以开启 kubectl 的命令补全功能了。但是，在实际使用的时候，Bash（包括 Linux 和 Mac 之间的差异）和 Zsh 有一些细节上的差异，下面是关于这些差异的相关说明。

### Bash on Linux

Bash 的命令补全脚本依赖[bash-completion](https://github.com/scop/bash-completion)项目，所以需要先安装该项目。

我们可以使用各种包管理器来安装bash-completion，例如：

```bash
$ sudo apt-get install bash-completion
# 或者
$ yum install bash-completion
```

安装完成后可以使用一下命令测试是否安装成功了：

```bash
type _init_completion
```

如果输出一段 shell 函数，则证明已经安装成功了，如果输出未找到的相关错误，则可以将下面的语句添加到你的`~/.bashrc`文件中去：

```bash
source /usr/share/bash-completion/bash_completion
```

> 是否必须将上面内容添加到`~/.bashrc`文件中，这取决于你使用的包管理工具，对于 apt-get 来说是必须的，yum 就不需要了。

一旦 bash-completion 安装完成后，我们就需要进行一些配置来让我们在所有的 shell 会话中都可以获取 kubectl 补全脚本。

一种方法是将下面的内容添加到~/.bashrc文件中：

```bash
source <(kubectl completion bash)
```

另一种方法是将 kubectl 补全脚本添加到/etc/bash_completion.d目录（如果不存在则新建）：

```bash
kubectl completion bash >/etc/bash_completion.d/kubectl
```

> `/etc/bash_completion.d`目录下面的所有补全脚本都会由 bash-completion 自动获取。

上面两种方法都是可行的，重新加载 shell 后，kubectl 的自动补全功能应该就可以正常使用了！

### Bash on MacOS

在 Mac 下面，就稍微复杂一点点了，因为 Mac 上的 Bash 默认版本是3.2，已经过时了，kubectl 的自动补全脚本至少需要 Bash 4.1 版本。

所以要在 Mac 上使用 kubectl 自动补全功能，你就需要安装新版本的 Bash，更新 Bash 是很容易的，可以查看这篇文章：[Upgrading Bash on macOS](https://itnext.io/upgrading-bash-on-macos-7138bd1066ba?gi=f728ffddf858)，这里我们就不详细说明了。

在继续向下之前，请确保你的 Bash 版本在 4.1 以上。和 Linux 中一样，Bash 的补全脚本依赖 bash-completion 项目，所以我们也必须要安装它。

我们可以使用 Homebrew 工具来安装：

```
brew install bash-completion@2
```

> @2代表 bash-completion v2 版本，kubectl 命令补全脚本需要 v2 版本，而 v2 版本至少需要 Bash 4.1 版本，所以这就是不能在低于4.1的 Bash 下面使用 kubectl 的补全脚本的原因。

brew 安装命令完成会输出一段提示信息，其中包含将下面内容添加到~/.bash_profile文件中的说明：

```
export BASH_COMPLETION_COMPAT_DIR=/usr/local/etc/bash_completion.d
[[ -r "/usr/local/etc/profile.d/bash_completion.sh" ]] && . "/usr/local/etc/profile.d/bash_completion.sh"
```

这样就完成了 bash-completion 的安装，但是建议将上面这一行信息添加到`~/.bashrc`当中，这样可以在子 shell 中也可以使用 bash-completion。

重新加载 shell 后，可以使用以下命令测试是否正确安装了 bash-completion:

```
type _init_completion
```

如果输出一个 shell 函数，证明安装成功了。然后就需要进行一些配置来让我们在所有的 shell 会话中都可以获取 kubectl 补全脚本。

一种方法是将下面的内容添加到~/.bashrc文件中：

```
source <(kubectl completion bash)
```

另外一种方法是添加 kubectl 补全脚本到/usr/local/etc/bash_completion.d目录下面：

```
kubectl completion bash >/usr/local/etc/bash_completion.d/kubectl
```

> 使用 Homebrew 安装 bash-completion 时上面的方法才会生效。

如果你还是使用 Homebrew 安装的 kubectl 的话，上面的步骤都可以省略，因为补全脚本已经被自动放置到/usr/local/etc/bash_completion.d目录中去了，这种情况，kubectl 命令补全应该在安装完 bash-completion 后就可以正常使用了。

最后，重新加载 shell 后，kubectl 自动提示应该就可以正常工作了。

### ZSH

Zsh 的补全脚本没有任何依赖，所以我们要做的就是设置让所有的 shell 会话中都可以自动补全即可。

可以在~/.zshrc文件中添加下面的内容来完成配置：

```
source <(kubectl completion zsh)
```

然后重新加载 shell 即可生效，如果重新加载 shell 后出现了compdef error错误，则需要开启compdef builtin， 可以在~/.zshrc文件的开头添加下面的内容来实现：

```
autoload -Uz compinit
compinit
```

## 快速查找资源

我们在使用 YAML 文件创建资源时，需要知道这些资源的一些字段和含义，一个比较有效的方法就是去 API 文档中查看这些资源对象的完整规范定义。

但是如果每次要查找某些内容的时候都切换到浏览器去查询也是很麻烦的一件事情，所以，kubectl 为我们提供了一个 `kubectl explain` 命令，可以在终端中直接打印出来所有资源的规范定义。kubectl explain命令的用法如下所示：

```bash
kubectl explain resource[.field]...
```

该命令可以输出请求的资源或者属性的一些规范信息，通过该命令显示的信息和 API 文档中的信息是相同的，参考下面使用示例：

![](/images/0011.svg)

默认情况下，`kubectl explain`命令只会显示属性的一级数据，我们可以使用`--recursive`参数来显示整个属性的数据：

```bash
kubectl explain deployment.spec --recursive
```

该命令会将 deployment.spec 属性下面所有的规范都打印出来。

如果你不太确定可以使用kubectl explain的资源名，可以使用下面的命令来获取所有资源名称：

```
kubectl api-resources
```

该命令会线上资源名称的复数形式（比如显示 deployments 而不是 deployment），还会显示一个资源的简写（比如 deploy），不过不用担心，我们可以用任意一个名称来结合kubectl explain命令使用的：

```bash
kubectl explain deployments.spec
# 或者
kubectl explain deployment.spec
# 或者
kubectl explain deploy.spec
```

## 使用自定义列格式化输出

`kubectl get`命令（读取集群资源）的默认输出格式如下：

```bash
$ kubectl get pods
NAME                                      READY   STATUS    RESTARTS   AGE
nginx-app-76b6449498-86b55                1/1     Running   0          23d
nginx-app-76b6449498-nlnkj                1/1     Running   0          23d
opdemo-64db96d575-5mhgg                   1/1     Running   2          23d
```

上面的输出结果是一种比较友好的格式，但是它包含的信息比较有限，比如上面只显示了 Pod 资源中的一些信息（与完整资源定义相比）。

所以这个时候就有[自定义输出格式](https://kubernetes.io/docs/reference/kubectl/overview/#custom-columns)的用武之地了，它允许我们自由定义要显示的列和数据，可以选择要在输出中显示为单独列的资源的任何字段。

自定义列输出的用法如下：`-o custom-columns=<header>:<jsonpath>[,<header>:<jsonpath>]...`

需要将每个输出列定义为`<header>:<jsonpath>`这样的键值对：

- `<header>`是列的名称，可以选择任何想要显示的内容。
- `<jsonpath>`是一个选择资源属性的表达式。

我们来看一个简单的例子：

```
$ kubectl get pods -o custom-columns='NAME:metadata.name'
NAME
nginx-app-76b6449498-86b55
nginx-app-76b6449498-nlnkj
opdemo-64db96d575-5mhgg
```

我们可以看到上面的命令输出了一个包含所有 Pod 名称的列。选择 Pod 名称的表达式是`metadata.name`，这是因为 Pod 的名称被定义在 Pod 资源的 `metadata` 字段下面的 name 字段中（我们可以在 API 文档或者使用`kubectl explain pod.metadata.name`命令来查看）。

现在假如我们要在输出结果中添加另外一列数据，比如显示每个 Pod 正在运行的节点，这时我们只需要向自定义列的选项中添加合适的列规范数据即可：

```bash
$ kubectl get pods \
  -o custom-columns='NAME:metadata.name,NODE:spec.nodeName'
NAME                                      NODE
nginx-app-76b6449498-86b55                ydzs-node2
nginx-app-76b6449498-nlnkj                ydzs-node1
opdemo-64db96d575-5mhgg                   ydzs-node2
```

节点名称的表达式是`spec.nodeName`，这是因为已调度 Pod 的节点信息被保存在了 Pod 的`spec.nodeName`字段中（可以通过`kubectl explain pod.spec.nodeName`查看）。

> 注意，Kubernetes 资源字段是区分大小写的。

我们可以通过这种方式将资源的任何字段设置为输出列的数据，只需要去查看资源规范并使用我们需要的任何字段即可！

但首先，我们还是来仔细来看看这些字段的选择表达式吧！

## JSONPath 表达式

选择资源字段的表达式是基于 JSONPath 的。

JSONPATH 是一种从 JSON 文件中提取数据的一种语言（类似于 XPath for XML）。选择单个字段只是 SJONPath 的最基本的用法，它还有很多其他的功能，比如列表选择器、过滤器等。

但是我们在使用kubectl explain命令的时候只支持部分 JSONPath 功能，下面我们用一些简单的示例来介绍下这些支持的功能：

```bash
# 选择一个列表的说有元素
$ kubectl get pods -o custom-columns='DATA:spec.containers[*].image'

# 选择一个列表的指定元素
$ kubectl get pods -o custom-columns='DATA:spec.containers[0].image'

# 选择和一个过滤表达式匹配的列表元素
$ kubectl get pods -o custom-columns='DATA:spec.containers[?(@.image!="nginx")].image'

# 选择特定位置下的所有字段（无论名称是什么）
$ kubectl get pods -o custom-columns='DATA:metadata.*'

# 选择具有特定名称的所有字段（无论其位置如何）
$ kubectl get pods -o custom-columns='DATA:..image'
```

另外一个非常重要的操作符是[]，Kubernetes 的资源很多字段都是列表，改操作符可以让我们选择这些列表中的一些元素，它通常与通配符[*]一起使用来选择列表中的所有元素。

## 示例演示







