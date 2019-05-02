---

title: Kubernetes 笔记
date: 2017-08-18T00:00:00.000Z
excerpt: 关于 Kubernetes 的一些零碎笔记
tags:
  - 架构
comments: true
redirect_from:
  - /2017/08/18/kubernetes-note.html
---


<!-- toc -->

- [API](#api)
  * [声明式的 API](#%E5%A3%B0%E6%98%8E%E5%BC%8F%E7%9A%84-api)
  * [API Response](#api-response)
  * [错误处理](#%E9%94%99%E8%AF%AF%E5%A4%84%E7%90%86)
  * [Resource Version](#resource-version)
  * [Version](#version)
  * [API Group](#api-group)
  * [Runtime config](#runtime-config)
  * [REGEX](#regex)
  * [字段格式](#%E5%AD%97%E6%AE%B5%E6%A0%BC%E5%BC%8F)
  * [PATCH 与 PUT](#patch-%E4%B8%8E-put)
- [Events](#events)
- [交互](#%E4%BA%A4%E4%BA%92)
  * [输出](#%E8%BE%93%E5%87%BA)
  * [输入](#%E8%BE%93%E5%85%A5)

<!-- tocstop -->

# API

## 声明式的 API

声明式： 结果是什么
命令式: 做什么

声明式的操作，相对于命令式操作，对于重复操作的效果是稳定的，这对于容易出现数据丢失或重复的分布式环境来说是很重要的。另外，声明式操作更容易被用户使用，可以使系统向用户隐藏实现的细节，隐藏实现的细节的同时，也就保留了系统未来持续优化的可能性

kubernetes 里的 API 都是声明式,我们描述好自己想要的 resource object,kubernetes 就会不断尝试去保证这个 resource object 按我们期望的方式存在.


## API Response

一般包含三部分

* metadata: 元数据
    * annotations: 一些元信息.给第工具用来存储和解析原信息用的.
    * labels: act as filter
    * namespace: resource 所处的 namespace
    * name: resource 名字
    * uuid: 唯一标识
    * creationTimestamp: 创建时间
    * deletionTimestamp: 计划删除的时间(graceful deletion)
    * resourceVersion: 每个 resource 的内部版本,可以用来确定是否发生了变化.也用于做并发控制
    * generation 
* spec: 具体描述,不同 resource 的属性不同。spec 里通过声明式的方式表明了期望的目标状态
* status: resource 的当前状态

下面以展示一下 kubernetes node api 的 metadata 作为样例:

**metadata**:

```yaml
metadata:
  annotations:
    flannel.alpha.coreos.com/backend-data: '{"VtepMAC":"9a:98:da:a1:b9:5d"}'
    flannel.alpha.coreos.com/backend-type: vxlan
    flannel.alpha.coreos.com/kube-subnet-manager: "true"
    flannel.alpha.coreos.com/public-ip: 172.18.0.4
    scheduler.alpha.kubernetes.io/taints: '[]'
    volumes.kubernetes.io/controller-managed-attach-detach: "true"
  creationTimestamp: 2017-07-13T09:45:11Z
  labels:
    beta.kubernetes.io/arch: amd64
    beta.kubernetes.io/os: linux
    ip: 172.18.0.4
    kubeadm.alpha.kubernetes.io/role: master
    kubernetes.io/hostname: 172.18.0.4
  name: 172.18.0.4
  resourceVersion: "3785002"
  selfLink: /api/v1/nodes172.18.0.4
  uid: f6fc3022-67af-11e7-b171-0017fa013946
```


可以看到 flannel 用它来存储了一些自己需要的信息.



这种区分几乎适用于 REST 架构中的大多数 resource.好处:
* 比一整个大的 body 结构清晰
* 模块化,每个部分的更新迭代不会影响整个 body 的结构
* 展示及处理方便


## 错误处理
kubernetes 的错误情况下的 response 也遵循上面相同的结构.示例如下:

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "pods \"grafana\" not found",
  "reason": "NotFound",
  "details": {
    "name": "grafana",
    "kind": "pods"
  },
  "code": 404
}
```

其特点是同时提供了机器可读(`reason`)以及人类可读(`message`)的消息. `reason`是对 http status code 的一种细化.



## Resource Version
上面所说 metadata 中的`resourceVersion`字段被 Kubernets 用来做并发控制,在更新一个 object 前,会去检查它的 resourceVersion 的值与之前存的值是否匹配,如果不匹配,则会抛出一个`StatusConflict`(http 409).



## Version
一般来说 REST API 都会要求有一个版本的概念,这样在架构迭代以及功能升级时可以用不同的版本来区分,保证语义及结构清晰.Kubernetes 做的更多,它也会用 version 来区分功能的成熟度(alpha,beta...)

* alpha
    * 可能包含 bug,默认关闭,不保证兼容性,适合测试使用
* beta
    * 大方向不会变,细节上可能会有修改.如果有不兼容的改动发生,会提供升级方式
* stable
    * 可稳定使用


## API Group
API 分组,一般 API 都会分为核心的对 Resource 进行操作的 API 以及其他零碎的 API.

一般 URL 格式为: `GROUP/VERSION`.GROUP 的名字建议的格式为 domin name 的格式,比如:
`widget.mycompany.com`    


## Runtime config
kubernets 支持非常多的参数(已经有很多人在网上吐槽了..),结合上面的版本以及分组,kube-api 有很多参数可以用来调控这些.比如禁掉一些 Group,禁掉一些 Version 的 API.一个完全可插拔的 API Server.


## REGEX
详细定义好各个 resource name 的`REGEX`要求,比如`namespace`,`service`中不能有点(`.`),只能小写(DNS 兼容).

## 字段格式
像 date,timestamp 等这种字段要保证在所有 API 中使用用一种格式

## PATCH 与 PUT
一般来说 PUT 应该发送 resource 的整个描述去 replace.PATCH 只发送需要更新的部分.但一般的场景中遵循此规则的应该不多,大多是只用一个 PUT,即用于整个更新也用于部分更新.

kubernetes 做的更多,在 PATCH 操作,它支持几种不同的操作语义

* Json Patch: Content-Type: application/json-patch+json
    * 定义了需要做的操作,示例如下:
        * `{"op": "add", "path": "/a/b/c", "value": [ "foo", "bar" ]}`
    
* Merge Patch: Content-Type: application/merge-patch+jso
    * objects 类的是合并,lists 类的是 replace
* Strategic Merge Patch: Content-Type: application/strategic-merge-patch+json
    * 支持各种自定义操作的 patch


# Events
时间也是系统设计里非常重要的一环.发生了上面,什么时间,谁操作的,结果是什么,不仅可以用系统内部问题的排查,也可以作为产品提供给其他用户.

Kubernetes 对一些重复性的时间做了累积,原本需要重复显示 N 次的事件现在用一个计数器来代替.这样可以减少数据存储和系统负载.


# 交互


## 输出

Unix 有一个非常出名的设计哲学:"一切皆文件",其优势在于，利用管道及其他工具，无数的小工具(`find`,`grep`,`awk`)等可以方便地协作以完成非常复杂的任务。各个工具均支持`plain text`作为输入以及输出。当然一切皆文件也有一个坏处，开源软件由不同的人写就，没有文本格式的约束，不同的工具均需要一定的文本格式的要求。工具越多，格式也越杂乱，需要人为记忆的东西也越多。例如很多 service 类工具的配置文件格式的差异，不同命令输出的差异等等

kubernetes 由很多组件构成，提供 API 的服务，命令行的访问工具(`kubetctl`)等等。类似于`docker cli`及`docker daemon`，`kubectl`也是从`kube-api-server`来读取信息展示给用户。我觉得 kubetcl 做的比较好的一点就是，它将数据的内部结构和展示二者分开了。示例如下:

获取节点的简单信息，不加任何参数:

```bash
[root@172 alauda]# kubectl get no
NAME         STATUS         AGE
172.18.0.4   Ready,master   38d
[root@172 alauda]#
```

指定输出 yaml 格式的详情

```yaml
...
spec:
  externalID: 172.18.0.4
  podCIDR: 10.1.0.0/24
  providerID: azure:////2882C846-5C01-BE47-B19A-4C5DB5F26348
status:
  addresses:
  - address: 172.18.0.4
    type: LegacyHostIP
  - address: 172.18.0.4
    type: InternalIP
  - address: 172.18.0.4
    type: Hostname
  allocatable:
    alpha.kubernetes.io/nvidia-gpu: "0"
    cpu: "16"
    memory: 57710432Ki
    pods: "110"
...
```


指定输出 json 格式的详情

```json
{
    "apiVersion": "v1",
    "kind": "Node",
    "metadata": {
        "annotations": {
            "flannel.alpha.coreos.com/backend-data": "{\"VtepMAC\":\"9a:98:da:a1:b9:5d\"}",
            "flannel.alpha.coreos.com/backend-type": "vxlan",
            "flannel.alpha.coreos.com/kube-subnet-manager": "true",
            "flannel.alpha.coreos.com/public-ip": "172.18.0.4",
            "scheduler.alpha.kubernetes.io/taints": "[]",
            "volumes.kubernetes.io/controller-managed-attach-detach": "true"
        },
        "creationTimestamp": "2017-07-13T09:45:11Z",
        "labels": {
            "beta.kubernetes.io/arch": "amd64",
            "beta.kubernetes.io/os": "linux",
            "ip": "172.18.0.4",
            "kubeadm.alpha.kubernetes.io/role": "master",
            "kubernetes.io/hostname": "172.18.0.4"
        },
        "name": "172.18.0.4",
        "resourceVersion": "3785243",
        "selfLink": "/api/v1/nodes172.18.0.4",
        "uid": "f6fc3022-67af-11e7-b171-0017fa013946"
    },
    "spec": {
        "externalID": "172.18.0.4",
        "podCIDR": "10.1.0.0/24",
        "providerID": "azure:////2882C846-5C01-BE47-B19A-4C5DB5F26348"
    },
    "....": "...."
}
```


默认的输出格式比较类似于 linux 上`ls`的默认输出。但也可以通过指定参数获取 json 或 yaml 格式的信息。前者适用于展示，后者适用于处理(管道).`ls`的输出目前既用于展示，也会用于输出。鉴于 json 目前的流行程度及简洁性，设想 plan9 上这些小工具均内置于对 json 或者 yaml 的支持，那么在利用管道及其他工具做数据处理的时候会大为简便，并且不易出错。


## 输入
kubectl 支持从一个描述文件里创建一个 resource(yaml 或 json 格式).因为 API 结构的一致,kubernetes 里面几乎所有的资源都可以通过一个`kubectl create -f`来创建出来. 因为每种resource都有大体相同的结构:
Kind,Version,Spec等等

