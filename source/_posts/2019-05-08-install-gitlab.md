---
title: 在 Kubernetes 中安装 Gitlab
toc: true
categories: 技术
date: 2019-05-08 13:36:33
excerpt: 一个详细的安装使用文档
tags:
    - Gitlab
    - Kubernetes
---

现在很少写这种安装类的博客了。之前在公司部署了一个 Gitlab 作为日常使用，因为步骤比较繁琐。在此把零零散散的资料汇聚一下，记录一个比较完整的安装过程


## 环境

* Kubernetes 1.13.1 三个高可用节点
* 每个节点上一个空余磁盘


## 步骤

### 安装 Rook

Rook 提供了基于 Ceph 的分布式存储，我们利用每个节点上的空余磁盘来支撑 Kubernetes 里的 PV/StorageClass 等


#### Helm 安装

首先，初始化磁盘

```bash
mkfs.ext4 /dev/vdb
mount /dev/vdb /var/lib/rook
mkdir /var/lib/rook
# TODO: add to /etc/fstab
```

然后通过 Helm 安装 Rook

```bash
helm repo add rook-stable https://charts.rook.io/stable
helm install --namespace rook-ceph-system rook-stable/rook-ceph
```

部署完成后可以看到`rook-ceph-system` Namespace 下运行的 Resource:

![](/images/gitlab-install/rook.png)


#### 创建 CephCluster

```yaml
#################################################################################
# This example first defines some necessary namespace and RBAC security objects.
# The actual Ceph Cluster CRD example can be found at the bottom of this example.
#################################################################################
apiVersion: v1
kind: Namespace
metadata:
  name: rook-ceph
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rook-ceph-osd
  namespace: rook-ceph
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rook-ceph-mgr
  namespace: rook-ceph
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rook-ceph-osd
  namespace: rook-ceph
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: [ "get", "list", "watch", "create", "update", "delete" ]
---
# Aspects of ceph-mgr that require access to the system namespace
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rook-ceph-mgr-system
  namespace: rook-ceph
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  verbs:
  - get
  - list
  - watch
---
# Aspects of ceph-mgr that operate within the cluster's namespace
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rook-ceph-mgr
  namespace: rook-ceph
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - services
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - jobs
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - delete
- apiGroups:
  - ceph.rook.io
  resources:
  - "*"
  verbs:
  - "*"
---
# Allow the operator to create resources in this cluster's namespace
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rook-ceph-cluster-mgmt
  namespace: rook-ceph
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: rook-ceph-cluster-mgmt
subjects:
- kind: ServiceAccount
  name: rook-ceph-system
  namespace: rook-ceph-system
---
# Allow the osd pods in this namespace to work with configmaps
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rook-ceph-osd
  namespace: rook-ceph
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: rook-ceph-osd
subjects:
- kind: ServiceAccount
  name: rook-ceph-osd
  namespace: rook-ceph
---
# Allow the ceph mgr to access the cluster-specific resources necessary for the mgr modules
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rook-ceph-mgr
  namespace: rook-ceph
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: rook-ceph-mgr
subjects:
- kind: ServiceAccount
  name: rook-ceph-mgr
  namespace: rook-ceph
---
# Allow the ceph mgr to access the rook system resources necessary for the mgr modules
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rook-ceph-mgr-system
  namespace: rook-ceph-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: rook-ceph-mgr-system
subjects:
- kind: ServiceAccount
  name: rook-ceph-mgr
  namespace: rook-ceph
---
# Allow the ceph mgr to access cluster-wide resources necessary for the mgr modules
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rook-ceph-mgr-cluster
  namespace: rook-ceph
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: rook-ceph-mgr-cluster
subjects:
- kind: ServiceAccount
  name: rook-ceph-mgr
  namespace: rook-ceph
---
#################################################################################
# The Ceph Cluster CRD example
#################################################################################
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    # For the latest ceph images, see https://hub.docker.com/r/ceph/ceph/tags
    image: ceph/ceph:v13.2.2-20181023
  dataDirHostPath: /var/lib/rook
  mon:
    count: 3
    allowMultiplePerNode: true
  dashboard:
    enabled: true
  storage:
    useAllNodes: true
    useAllDevices: false
    config:
      databaseSizeMB: "1024"
      journalSizeMB: "1024"
```

这个 yaml 列表包含了如下的 Resource:

1. ns/rook-ceph
2. sa/rook-ceph/rook-ceph-mgr
3. role/rook-ceph/rook-ceph-osd
4. role/rook-ceph/rook-ceph-mgr-system
5. role/rook-ceph/rook-ceph-mgr
6. rolebinding/rook-ceph/rook-ceph-cluster-mgmt
7. rolebinding/rook-ceph/rook-ceph-osd
8. rolebinding/rook-ceph/rook-ceph-mgr-system
9. rolebinding/rook-ceph/rook-ceph-mgr-cluster
10. cephecluster/rook-ceph/rook-ceph

Helm Rook 里已经定义好了相关的 CRD,我们再这里就是要创建一个 CephCluster 的资源，可以看到它的 YAML 里指定了之前我们初始化好的`/var/lib/rook`目录。

```bash
kubectl create -f cluster.yaml
```

#### 创建 StorageClass

```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block
provisioner: ceph.rook.io/block
parameters:
  blockPool: replicapool
  # The value of "clusterNamespace" MUST be the same as the one in which your rook cluster exist
  clusterNamespace: rook-ceph
  # Specify the filesystem type of the volume. If not specified, it will use `ext4`.
  fstype: xfs
# Optional, default reclaimPolicy is "Delete". Other options are: "Retain", "Recycle" as documented in https://kubernetes.io/docs/concepts/storage/storage-classes/
reclaimPolicy: Retain
```


#### 页面

Rook 自带了一个 UI，我们可以通过 Ingress 配置的地址来访问和管理 Ceph 集群:

![](/images/gitlab-install/ceph.png)



### 安装 Cert Manager
我们后面需要通过 HTTPS 访问 Gitlab/Rook 等，Cert Manager 与 Ingress 一起提供这个功能。


#### Helm 安装

```bash
wget https://github.com/helm/charts/blob/master/stable/cert-manager/cert-manager-v0.6.0-dev.5.tgz
helm install --name=cert-manager --namespace=kube-system ./cert-manager-v0.6.0-dev.5.tgz
```

#### 创建 ClusterIssuer

ClusterIssuer 是 Cert Manager 提供的 CRD


```yaml
apiVersion: v1
items:
- apiVersion: certmanager.k8s.io/v1alpha1
  kind: ClusterIssuer
  metadata:
    creationTimestamp: 2018-12-24T07:30:04Z
    generation: 1
    name: letsencrypt-prod
    namespace: ""
    resourceVersion: "2346374"
    selfLink: /apis/certmanager.k8s.io/v1alpha1/clusterissuers/letsencrypt-prod
    uid: bb17c08d-074d-11e9-9e39-525400f36999
  spec:
    acme:
      email: <your-email>
      http01: {}
      privateKeySecretRef:
        key: ""
        name: letsencrypt-prod
      server: https://acme-v02.api.letsencrypt.org/directory
  status:
    acme:
      uri: https://acme-v02.api.letsencrypt.org/acme/acct/48274855
    conditions:
    - lastTransitionTime: 2018-12-24T07:30:09Z
      message: The ACME account was registered with the ACME server
      reason: ACMEAccountRegistered
      status: "True"
      type: Ready
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

中间的`<you-email>`需要替换成具体的 Email。




### 安装 Gitlab


#### Helm 安装

Gitlab 也是需要通过 Helm 安装，但是需要自定义的变量比较多。

以下是 values 自定义的部分

```yaml
redisImage: redis:3.2.10
redisDedicatedStorage: true
redisStorageSize: 5Gi
postgresImage: postgres:9.6.3
postgresStorageSize: 30Gi
gitlabRailsStorageSize: 70Gi
gitlabRegistryStorageSize: 70Gi
gitlabConfigStorageSize: 1Gi
baseDomain: <domain>
baseIP: <ip>
legoEmail: <your-email>
gitlabConfigStorageClass: rook-ceph-block
gitlabDataStorageClass: rook-ceph-block
gitlabRegistryStorageClass: rook-ceph-block
postgresStorageClass: rook-ceph-block
redisStorageClass: rook-ceph-block
gitlab: CE
gitlabCEImage: gitlab/gitlab-ce:11.6.5-ce.0
gitlabRunnerImage: gitlab/gitlab-runner:alpine-v11.6.1
runners:
  privileged: true
gitlab-runner:
  image: gitlab/gitlab-runner:alpine-v11.6.1
```

说明:

1. baseDomain/baseIP/legoEmail: 这几个需要根据具体情况替换
2. *Size: 这些变量的值就是我们安装的 Gitlab 所需要使用的空间，当然都是越大越好

Helm 安装:

```bash
helm install --name gitlab -f values.yaml gitlab/gitlab-omnibus --namespace=gitlab
```


#### 修改 Ingress
之前已经安装了 Cert Manger,我们可以通过给 Gitlab 的 Ingress 添加如下的 Annotation 来支持 HTTPS

```bash
# kubectl edit ingress gitlab-gitlab -n gitlab
'certmanager.k8s.io/cluster-issuer: letsencrypt-prod'
```


我们可以通过 Ingress 看到，有以下四个域名可用了：

* gitlab.aks.myalauda.cn: 主页面
* registry.aks.myalauda.cn: 镜像仓库
* mattermost.aks.myalauda.cn: 一个团队协作工具
* prometheus.aks.myalauda.cn: 监控信息

#### 权限相关

```yaml
kind: ClusterRoleBinding
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   name: gitlab-cluster-admin
 subjects:
 - kind: ServiceAccount
   name: gitlab
   namespace: gitlab
 roleRef:
   kind: ClusterRole
   name: cluster-admin
   apiGroup: rbac.authorization.k8s.io
```
增加 Gitlab 的权限，不然可能会有未知的访问问题


#### Gitlab Runner 配置
让 Gitlab Runner 支持 docker-in-doker

```bash
cat >> /etc/gitlab-runner/config.toml << EOF
          [[runners.kubernetes.volumes.host_path]]
            name = "docker"
            path = "/var/run/docker.sock"
            mount_path = "/var/run/docker.sock"
            read_only = false
    EOF
```

这个是通过修改 configmaps/gitlab/gitlab-gitlab-runner 实现:

![](/images/gitlab-install/gitlab-runner-config.png)
 
 



### 安装 SonarQube

SonarQube 可以提供对多种语言的代码静态分析，它自身是一个 Server，带页面，也支持各种插件。与 Gitlab 的集成是通过 Gitlab Account 以及 Token 等方式实现。

#### YAML 安装

以下是相关 YAML

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sonar-data
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: rook-ceph-block
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sonar-extensions
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: rook-ceph-block
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres
type: Opaque
data:
  password: NDl1ZjNtenMxcWR6NXZnbw==
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: sonarqube
  name: sonarqube
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: sonarqube
    spec:
      containers:
        - name: sonarqube
          image: sonarqube:7.1
          resources:
            requests:
              cpu: 500m
              memory: 1024Mi
            limits:
              cpu: 2000m
              memory: 2048Mi
          volumeMounts:
          - mountPath: "/opt/sonarqube/data/"
            name: sonar-data
          - mountPath: "/opt/sonarqube/extensions/"
            name: sonar-extensions
          env:
          - name: "SONARQUBE_JDBC_USERNAME"
            value: "gitlab"
          - name: "SONARQUBE_JDBC_URL"
            value: "jdbc:postgresql://gitlab-gitlab-postgresql.gitlab/sonar"
          - name: "SONARQUBE_JDBC_PASSWORD"
            valueFrom:
              secretKeyRef:
                name: postgres
                key: password
          ports:
          - containerPort: 9000
            protocol: TCP
      volumes:
      - name: sonar-data
        persistentVolumeClaim:
          claimName: sonar-data
      - name: sonar-extensions
        persistentVolumeClaim:
          claimName: sonar-extensions
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: sonarqube
  name: sonarqube
spec:
  ports:
    - name: sonar
      port: 80
      protocol: TCP
      targetPort: 9000
  selector:
    app: sonarqube
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    certmanager.k8s.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
  name: sq
spec:
  rules:
  - host: <sonarqube-host>
    http:
      paths:
      - backend:
          serviceName: sonarqube
          servicePort: 80
  tls:
  - hosts:
    -  <sonarqube-host>
    secretName: sonar
status:
  loadBalancer:
    ingress:
    - {}
```

包含如下资源:

1. 两个 PVC 分别用于存放 SonarQube 自身的数据以及插件的数据
2. Service/Ingress: 这部分类似于 Gitlab 自身的配置

需要注意的是，因为 SonarQube 自身需要 DB,而我们在 Gitlab 安装的时候已经部署了 DB,所以可以继续复用

以上资源最好也都单独放在一个 NS 下面，比如


```yaml
kubectl apply -f sq.yaml -n sq
```

结果如图所示：

![](/images/gitlab-install/sq.png)


#### DB 配置

在 Gitlab 的 DB 中创建 sonar DB

```sql
CREATE DATABASE sonar WITH OWNER gitlab;
```

数据库的名字需要与上面 SonarQube 的 Deployment YAML 中的 JDBC 地址一致。


#### 安装插件
需要安装一下几个插件

* bitbucket oauth: 用于支持 Bitbucket 登录
* git: git 支持
* go: Golang 语言的分析支持
* gitlab: 与 Gitlab 集成

我们可以通过 SonarQube 的页面安装，也可以直接用 wget 下载到相应的目录中。如下的两个 Gitlab 插件就只能通过下载安装：

```bash
# kubectl exec to sonarqube pod
wget https://github.com/gabrie-allaigre/sonar-auth-gitlab-plugin/releases/download/1.0.0/sonar-auth-gitlab-plugin-1.0.0.jar
wget https://github.com/gabrie-allaigre/sonar-gitlab-plugin/releases/download/4.0.0/sonar-gitlab-plugin-4.0.0.jar
```


#### 设置 Bitbucket 登录

之前已经配置过 Bitbucket 的 OAuth 信息，这里可以复用。需要在 SonarQube 配置好以下参数:

```bash
sonar.core.serverBaseURL 
```


## 集成

### Bitbucket
通过 OAuth 的方式与 Bitbucket 集成，首先需要在 Bitbucket 的页面配置相关的 id/secret 等，然后更新 gitlab-gitlab 的 Deployment:

```bash
gitlab_rails['omniauth_providers'] = [{"name" => "bitbucket","app_id" => "<id>", "app_secret" => "<token>","url" => "https://bitbucket.org/"}]
```

![](/images/gitlab-install/gitlab-deploy-config.png)

如图所示，Gitlab 将大部分配置存储于其 Deployment 的一个 ENV 中，各种自定义配置都可以通过修改这个环境变量实现。



### SonarQube

#### 用户配置

我们需要在 Gitlab 添加一个 SonarQube 的用户，并赋予相应的权限，这样他可以在 MR 上添加分析的结果。
同时也要在 SonarQube 上添加一个 Gitlab 用户，用于上报分析结果给 SonarQube。

#### 项目配置

在实际的项目中，我们需要有一个配置文件，主要用来指明一些具体的分析规则，如下是一个示例

```text
sonar.projectKey=<project_name>
sonar.projectName=<project_name>
sonar.projectVersion=0.1
sonar.login=<username>
sonar.password=<token>
sonar.scm.provider=git
# GoLint report path, default value is report.xml
# TODO: add back
# sonar.golint.reportPath=report.xml
# Cobertura like coverage report path, default value is coverage.xml
sonar.typescript.lcov.reportPaths=lcov.info
# if you want disabled the DTD verification for a proxy problem for example, true by default
sonar.coverage.dtdVerification=false
# JUnit like test report, default value is test.xml
sonar.test.reportPath=test-reports/junit.xml
sonar.sources=.
sonar.test=.
sonar.test.inclusions=./**/*_test.go
sonar.test.exclusions=./vendor/**
sonar.go.coverage.reportPaths=coverage.out
# Ignore duplicating string literal issue
sonar.issue.ignore.multicriteria=e1
sonar.issue.ignore.multicriteria.e1.ruleKey=go:S1192
```

需要注意的地方:

1. project_name: 这个需要在 SonarQube 的页面预先创建好对应的项目
2. username/token: 配置的用于访问 SonarQube 的用户和 Token


#### CI 配置
在 CI 中，我们可以专门增加一个 Stage 用于代码分析:

```yaml
sonarqube:
  only:
    - merge_requests
  stage: analysis
  image: ciricihq/gitlab-sonar-scanner
  dependencies:
    - unit-test
  variables:
    SONAR_URL: <url>
    SONAR_ANALYSIS_MODE: preview
    SONAR_GITLAB_PROJECT_ID: <project_id>
  script:
    - gitlab-sonar-scanner

sonarqube-reports:
  only:
    - master
  stage: analysis
  image: ciricihq/gitlab-sonar-scanner
  variables:
    SONAR_URL:  <url>
    SONAR_ANALYSIS_MODE: publish
  script:
    - gitlab-sonar-scanner

```
其中`url`指 SonarQube 的访问地址, `project_id`指 Gitlab 项目的 ID，一般用 `<group>/<project>`即可。
上面的两个 Job 作用不同，`sonarqube`用于分析并且添加 Comment, `sonarqube-reports`用于分析并且上报结果到 SonarQube。

最终效果如下:

![](/images/gitlab-install/sq-comment.png)

如果有错误，Comment 中也会详细列出来并可以跳转到具体的 SonarQube 页面查看。


### 邮箱

Gitlab 发送邮箱的部分也是需要自己配置，和上面对 Bitbucket 的 OAuth 配置一样，直接 edit 其 Deployment 即可。示例配置如下：

```yaml
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.office365.com"
gitlab_rails['smtp_port'] = 587
gitlab_rails['smtp_user_name'] = "<email>"
gitlab_rails['smtp_password'] = "<password>"
gitlab_rails['smtp_domain'] = "<domain>"
gitlab_rails['gitlab_email_from'] = '<email>'
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_tls'] = false
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_openssl_verify_mode'] = 'peer'
```


### JIRA
Gitlab 目前与 Jira 的关联还比较弱，我们可以在 Gitlab 的 Service Templates 里配置好具体的 JIRA 地址，然后在 Commit Message 里关联上相关的 JIRA ID。这里也需要在 JIRA 上创建相应的 Gitlab 用户，这样我们提交的 MR 与 JIRA 的关联就可以在 JIRA 的页面自动展示出来。

![](/images/gitlab-install/jira.png)


当然另一方面，我们也可以利用 Webhook + JIRA SDK，来实现一个 WebHook，通过在 Commit Message 里增加相应的动作来驱动 JIRA 状态的变更，这个可以根据具体的业务情况用 Python 脚本很容地实现。

### Danger
请见本站关于 Danger 与 Gitlab 的集成相关的文章

TODO: 增加 Link


## 使用

### LGTM

Gitlab CE 因为砍掉了很多 EE 版本的功能，没有像 GITHUB 那样的通过 `/lgtm`或者`LGTM`这样的关键词来自动合并 MR。这个也可以通过部署一个简单的 Server 来作为 Webhook 实现。再通过增加一个`MergeBot`用户来执行合并的动作，能够极大地简化我们的代码合并流程。


### CI/CD

#### 敏感信息
CI/CD 中要使用很多 password/token 等敏感信息，直接放在 CI 文件中并且保存在 Git 仓库中是非常不安全的，Gitlab 的 CI 提供了 Pipeline 的环境配置信息，将敏感信息存储在 Gitlab，可以在 ci 文件中直接引用:

![](/images/gitlab-install/ci-secret.png)



### Badges
Badges 的配置是在项目上的，不用在项目的 Readme 等文件中插入什么信息。这点是比较好的。目前 CE 版本支持 Build 状态以及单元测试覆盖率等，一般情况下是够用的。也可以增加第三方的 Badge 地址.

![](/images/gitlab-install/badge.png)


### CLI 直接创建 MR

Gitlab CE 的比较新的版本里已经支持从命令行 Push 的时候直接创建 Merge Request 了，这也是一个非常好用的功能。最简单的方式就是在 Push 后面增加参数:

```bash
git push origin <branch> -o merge_request.create
```

其他的参数可参考官方文档使用。


### 升级

因为 Gitlab 的功能更新比较频繁，我们可以经常性地更新 Gitlab 的版本，也是通过直接 edit deployment，更新镜像版本字段即可。当然因为是单实例的服务，会有几分钟的服务中断，影响也不大。需要注意的是 runner 的版本也需要同步更新。


![](/images/gitlab-install/version.png)



## Links
1. [GitLab CI/CD Pipeline Configuration Reference](https://docs.gitlab.com/ce/ci/yaml/)
2. [Python Jira Library](https://github.com/pycontribs/jira)
3. [Gitlab LGTM Bot](https://github.com/hjanuschka/GitlabBot)





