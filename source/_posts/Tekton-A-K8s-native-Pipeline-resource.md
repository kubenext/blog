---
title: 'Tekton, A K8s-native Pipeline resource'
date: 2019-10-01 16:06:19
top: true 
categories: Kubernetes
tags: 
- Kubernetes
- Tekton
---

## Tekon 入门

Tenton主要由以下五个核心概念组成：

- Task
- TaskRun
- Pipeline
- PipelineRun
- PipelineResource

这五个概念都是以CRD的形式提供服务的，以下分别简述以下这五个概念的含义。

### Task

Task就是一个任务执行模版，之所有说Task是一个模版是因为Task定义中可以包含变量，Task在正在执行的时候需要给定变量的具体值。Tokon的Task很类似于一个函数的定义，Task通过input.params定义需要哪里入参，并且每一个入参还可以指定默认值。Task的steps字段表示当前Task是由哪些子不步骤组成的。每一个步骤具体就是一个镜像的执行，镜像的启动参数可以通过Task的入参使用模版语法进行配置。

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: task-with-parameters
spec:
  inputs:
    params:
      - name: flags
        type: array
      - name: someURL
        type: string
  steps:
    - name: build
      image: maven:3
      command: ["mvn","pacage"]
      args: [ "echo ${inputs.params.flags}; echo ${someURL}" ]
```

### TaskRun

Task定义好以后是不能执行的，就像一个函数定义好后需要调用才能执行一样。所有需要再定义一个TaskRun去执行Task。TaskRun主要是负责设置Task需要的参数，并通过taskRef字段引用要执行的Task。

```yaml
apiVersion: tekton.dev/v1alpha1
kind: TaskRun
metdata:
  name: run-with-parameters
spec:
  taskRef:
    name: task-with-paramters
  inputs:
    params:
      - name: flags
        value: "--set"
      - name: someURL
        value: "https://githube.com"
```

### Pipeline

一个TaskRun只能执行一个Task，当需要编排多个Task的时候需要用到Pipeline了。Pipeline是一个编排Task的模板。Pipeline的params声明的顺序依次执行。Pipeline在片拍Task的时候需要给每一个Task、传入必须的参数，这些参数的值可以来自Pipeline自身的params。

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: pipeline-with-parameters
spec:
  params:
    - name: context
      type: string
      description: Path to context
      default: /some/where/or/other
  tasks:
    - name: task-1
      taskRef:
        name: build
      params:
        - name: pathToDockerFile
          value: Dockerfile
        - name: pathToContext
          value: "${params.context}"
    - name: task-2
      taskRef:
        name: build-push
      runAfter:
        source-to-image
      params:
        - name: pathToDockerFile
          value: Dockerfile
        - name: pathToContext
          value: "${params.context}"
```

### PipelineRun

和Task一样Pipeline定义完成以后也是不能直接执行的，需要PipelineRun才能执行Pipeline。PipelineRun的主要作用是给Pipeline设定必要的入参，并执行Pipeline。

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  name: pipelinerun-with-parameters
spec:
  pipelineRef:
    name: pipeline-with-parameters
  params:
    - name: "context"
      value: "/path"
```

### PipelineResource

前面已经介绍了Tekton的四个核心概念。现在我们已经知道怎么定义task、执行task以及编排task了。但可能你还想在task之间共享资源，这就是PipelineResource的作用。比如我们可以把git仓库信息放在PipelineResource中。这样所有的Pipeline就可以共享这些数据了。

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: wizzbang-git
  namespae: default
spec:
  type: git
  params:
    - name: url
      value: https://github.com/ex.git
    - name: revision
      value: master
```

### 授权信息

git仓库、镜像仓库这些都是需要授权才能使用的。所有还需要一种授权信息的机制。Tekton本身是Kubernetes原生的编排系统。所有可以直接使用Kubernetes的ServiceAccount机制实现授权。

- 定义一个保存镜像仓库授权信息的Secret

```yaml
apiVersion: v1
kind: Secret
metdata:
  name: ack-cr-push-secret
  annotations:
    tekon.dev/docker-0: https://registry.ex.com
type: kubernetes.io/basic-auth
stringData:
  username: <cleartext non-encoded>
  password: <cleartext non-encoded>
```

- 定义ServiceAccount，并使用上面的secret

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline-account
secrets:
- name: ack-cr-push-secret
```

- PipelineRun 中引用ServiceAccount
  
```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
medata:
  generateName: tekton-kn-sample
spec:
  pipelineRef:
    name: build-and-deploy-pipline
    ...
  serviceAccount: pipeline-account
```