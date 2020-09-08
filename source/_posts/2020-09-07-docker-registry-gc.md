---
title: Docker 私有 Registry 的镜像 GC 问题
toc: true
excerpt: ...
categories: 技术
date: 2020-09-07 18:13:47
tags: [Docker,Kubernetes]
---

## 需求 

一个轻量级的 docker registry

## 方案讨论

Harbor 的功能丰富，但是过于重量级了，与其他平台集成并不方便。Portus 类似。这一类产品都比较侧重 UI 和认证，一般都带有数据库， 与
K8S 的集成也比较麻烦。最终看起来还是 registry 最合适，不过就是功能太简陋了。需要考虑的东西很多:

1. HTTPS 
2. 暴露方式 -> Ingress/NodePort. 最好的方式当然是 Ingress, 但虽然 registry 支持 SubPath, 但
docker client 不支持。所以只能退而求其次用 NodePort. 对外地址的暴露只能设计 CR 来做了
3. GC -> 这块看起来都受限于 registry 本身的能力。它提供了命令行工具能手工清除 blob, 但其他的 metadata 没删完。
导致做完 GC 还得重启下 registry. 同时还得保持 GC 时 readonly.

这里面最繁琐的就是 GC, 目前提供的工具非常的原始。harbor 集成了 gc 的功能，应该是在 admin 页面
提供了按钮来手动触发。那么在 K8S + registry 的限定范围内应该怎么做，目前想好的流程是：

1. 在 registry 的 deployment 里加一个 container 来处理 GC， 这样的好处是可以共享配置，存储和 PID
2. gc container 和 registry container 共用同样的镜像，但 CMD 不一样
3. gc container 的 CMD 是一个 while 循环，用来周期性地跑 GC 命令
4. 跑完 GC 之后，如果确实清理了一些 blob, 还需要重启 registry. 不然因为其他一些 metadata 没有清理， 直接 push
和 pull 会出错。

这样就组成了一个低成本的可用的 GC 方案。缺点就是没法方便地将 registry 置为  readonly. 理论上来讲,
gc 的周期设置长一点，问题不大. 

## 遇到的问题
主要的一个问题就是 registry 的 gc 命令有 bug, 在配置识别上做的不好，默认配置好了 registry 
data 的地址还是一直扫描不到。不得已用了 [docker-distribution-pruner](https://gitlab.com/gitlab-org/docker-distribution-pruner)
这个库，它做的事情跟 gc 比较类似，只是 outpout 更易读一些。算是能解决 registry gc command 的
配置识别问题，但它自己也是因为比较 alpha 的成熟度，也有 bug. 比如目前发现 registry 里没有内容的
时候直接就抛异常了，但这个问题不大。因为我们是 loop gc的，等到有数据了自然就正常了。再不济 fork 代码
很容易 fix.

## 配置示例

首先 gc container 所用镜像的 Dockerfile

```Dockerfile
FROM registry:2.7.1

ADD docker-distribution-pruner /
ADD run.sh /run.sh

RUN chmod a+x /run.sh

CMD ["/bin/sh", "-c", "/run.sh"]%
```

```bash
#!/bin/sh

while true
do
  # run gc, if delete something, restart registry process
  sleep 120
  EXPERIMENTAL=true /docker-distribution-pruner -config=/etc/docker/registry/config.yml -delete  2>&1 | grep -w "DELETE" && pkill /bin/registry
  sleep $GC_PERIOD
done%
```

逻辑上就是不断地调用 docker-distribution-pruner 来做清理，如果发现有清理的数据，则 kill 掉 registry
的进程。k8s 会自动重启。因为我们默认是用 hostpath. 如果直接是用重新调度 pod 的方式会重启，数据直接就没了。
所以还是倾向于尽量原地重启。

```yaml
      containers:
        - name: gc
          image: docker-registry-pruner:v1.0
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: "RELEASE-NAME-docker-registry-config"
              mountPath: "/etc/docker/registry"
            - name: data
              mountPath: /var/lib/registry/
          env:
            - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
              value: "/var/lib/registry"
            - name: GC_PERIOD
              value: "604800"
```

这部分就是 gc container 在 Deployment 里的配置。 它挂在的 volume 与 registry 的 Container 是类似的。
关于默认 registry 的 配置，使用的是 helm 官方的 chart.







