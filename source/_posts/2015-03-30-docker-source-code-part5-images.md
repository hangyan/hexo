---
layout: post
title: docker 源码分析(5) -- 镜像拉取及存储
date: 2015-03-30 21:37:39 +0800
tags: [docker]
imagefeature:
comments: true
share: true
excerpt: ""
description: "docker images pulling and store"
---

本篇的内容主要是关于`docker`镜像的。在我们安装好`docker`之后，要想使用它，第
一步就是要下载一些镜像。本文将依据此流程分析`docker`中镜像的拉取、存储等相关内容。

<!--more-->

## 简介
`docker`中几乎所有的操作都是通过`WEB API`的方式执行的，所以当我们在命令行下敲下
`docker pull`或者通过`Docker Remote API`来拉取镜像时，`docker`便准备好各项参数，
开始向内部的`web server`发送`http`请求，最终由提前注册好的`Handlers`来执行相关
操作。我们将按照这个步骤来逐步分析与镜像拉取,存储相关的源码。


## 子命令执行
我们仍从`docker`的`main`函数入口处开始。在
[docker 主程序分析](http://hangyan.github.io/docker/docker-source-code-part2-start-and-args/)
中里面我已经提到，如果没有`-d`参数，最终便当作`client`对待并且将参数当作子命令来
解析执行,代码如下:


```go
if err := cli.Cmd(flag.Args()...); err != nil {
    if sterr, ok := err.(*utils.StatusError); ok {
        if sterr.Status != "" {
	        log.Println(sterr.Status)
        }
        os.Exit(sterr.StatusCode)
    }
    log.Fatal(err)
}
```

比如我在命令行下敲下`docker pull ubuntu`这个命令，那么在`Cmd`函数中，首先要做的
便是找到与`pull`相对应的`handler`.在这里并不是像一般的做法那样通过提前注册好的
`map`来查找，而是直接进行一些字符串转换，比如`pull`对应的叫`CmdPull`,`push`对应
的叫`CmdPush`,规则就是首字母大写并且加上一个`Cmd`前缀.

`github.com/docker/docker/api/client/cli.go`:
```go
func (cli *DockerCli) getMethod(args ...string) (func(...string) error, bool) {
    camelArgs := make([]string, len(args))
    for i, s := range args {
        if len(s) == 0 {
            return nil, false
        }
        camelArgs[i] = strings.ToUpper(s[:1]) + strings.ToLower(s[1:])
    }
    methodName := "Cmd" + strings.Join(camelArgs, "")
    method := reflect.ValueOf(cli).MethodByName(methodName)
    if !method.IsValid() {
        return nil, false
    }
    return method.Interface().(func(...string) error), true
}
```


所以我们可以在`github.com/docker/docker/api/client/commands.go`文件中见到很多个
这样的函数:

![ ][13]

## 发送请求
找到了执行函数，下面就是将其他参数传进去并开始执行，我们来看下`CmdPull`的流程:

我们在拉取镜像时,参数中的`image name`可以是多种多样的，有以下几类：

1. 只有名字 比如`ubuntu`
2. 名字和 tag 比如`ubuntu:14.04`
3. 命名空间,名字,(tag) 比如`tutum/redis`或者后面加个`tag`
4. 前面有私有仓库地址 比如 `127.0.0.1:5000/ubuntu:14.04`
...

所以我们既要检测参数中是否包含非法字符，也要对这各种情况解析出正确的`host`地址
和`name`,`tag`.不过了解了其结构之后，解析的代码就显得简单多了，不再具
体分析.

```go
taglessRemote, tag := parsers.ParseRepositoryTag(remote)
if tag == "" && !*allTags {
    newRemote = taglessRemote + ":" + graph.DEFAULTTAG
}

if tag != "" && *allTags {
    return fmt.Errorf("tag can't be used with --all-tags/-a")
}
```

注意其中的`DEFAULTTAG`，其值为`latest`,所以如果我们`pull`镜像时没有指定`tag`并且
`--all-tags`为`false`，则只会拉取`tag`为`latest`的镜像。


参数中的`hostname`部分需要单独解析出来，因为有安全认证的考虑.需要读取相关的配置
文件，并解析参数，最终要作为`http header`中的参数发送出去.

```go
hostname, _, err := registry.ResolveRepositoryName(taglessRemote)
if err != nil {
	return err
}

cli.LoadConfigFile()

authConfig := cli.configFile.ResolveAuthConfig(hostname)
```

需要注意的是,如果参数中没有指定`hostname`，则默认是从官方仓库拉取镜像,其值由变量
`INDEXSERVER`定义: `https://index.docker.io/v1/`。


参数解析好之后，便可以执行`http`请求了。`docker pull`所发起的是`POST`请求,地址为
`/iamges/create`:

```go
v  = url.Values{}

v.Set("fromImage", newRemote)

pull := func(authConfig registry.AuthConfig) error {
    buf, err := json.Marshal(authConfig)
    if err != nil {
        return err
    }
    registryAuthHeader := []string{
        base64.URLEncoding.EncodeToString(buf),
    }
        
    return cli.stream("POST", "/images/create?"+v.Encode(), nil, cli.out, map[string][]string{
        "X-Registry-Auth": registryAuthHeader,
    })
}

if err := pull(authConfig); err != nil {
    ...
}
```

在`web server`端，我们可以查到`/images/create`所对应的`handler`名称。

`github.com/docker/docker/api/server/server.go` : 

![ ][1]

所以下面我们就开始分析`postImageCreate`的执行流程.


## 请求处理

之前提到过，`daemon`将各种操作统一为`job`的形式，镜像的创建也不例外。所以`postImageCreate`
的主要工作即是进行`job`的相关环境的设定：

```go
image = r.Form.Get("fromImage")
repo  = r.Form.Get("repo")
tag   = r.Form.Get("tag")


if image != "" { //pull
    if tag == "" {
        image, tag = parsers.ParseRepositoryTag(image)
    }
    metaHeaders := map[string][]string{}
    for k, v := range r.Header {
        if strings.HasPrefix(k, "X-Meta-") {
            metaHeaders[k] = v
        }
    }
    job = eng.Job("pull", image, tag)
    job.SetenvBool("parallel", version.GreaterThan("1.3"))
    job.SetenvJson("metaHeaders", metaHeaders)
    job.SetenvJson("authConfig", authConfig)
}
```


需要注意的是,`postImageCreate`对应了两个`subcommand`，分别是`pull`和`import`,都
是用来创建镜像的，所以它们`post`的地址一样。二者通过传递不同的参数来区分创建不同
的`job`,具体就是`fromImage`这个参数.在`CmdPull`中，我们设置了这个参数，但是在`CmdImport`
中，则设置了其他的参数,二者流程类似，所以本文只对`pull`的流程做深入解析.

我们可以从`github.com/docker/docker/graph/service.go`文件中查到名字为`pull`的
`job`对应的`handler`:

![ ][2]

到了这个`CmdPull`（不要与前面的`CmdPull`搞混),才真正开始镜像拉取的过程。


## 镜像拉取

### 冲突检测
大家可能有过这样的经验，在一个终端下执行`docker pull`,时间太长，也不确定是不是成
功了，就`Ctrl-C`掉,然后重开一个终端重新执行，这时候就会提示已经有一个`client`在
拉取了，需要等待。这就是`CmdPull`最开始做的事情：确保只有一个`client`在拉取同一
个镜像:

```go
c, err := s.poolAdd("pull", localName+":"+tag)
if err != nil {
    if c != nil {
        job.Stdout.Write(sf.FormatStatus("", "Repository %s already being pulled by another client. Waiting.", localName))
        <-c
        return engine.StatusOK
    }
    return job.Error(err)
}
defer s.poolRemove("pull", localName+":"+tag)
```

`pollAdd`可以对`pull`和`push`两个过程进行检测(通过第一个参数),然后通过一个`map`
确认是否已经有`client`在拉取第二个参数标识的镜像。


### 名称解析
`CmdPull`传入的参数有两个:`image`和`tag`。`image`是包含仓库地址的，我们需要在拉取
之前将其解析出来，建立连接,并对镜像名做进一步规范化处理:

```go
hostname, remoteName, err := registry.ResolveRepositoryName(localName)

endpoint, err := registry.NewEndpoint(hostname, s.insecureRegistries)

r, err := registry.NewSession(authConfig, registry.HTTPRequestFactory(metaHeaders), endpoint, true)

if endpoint.VersionString(1) == registry.IndexServerAddress() {
    localName = remoteName

    isOfficial = isOfficialName(remoteName)
    if isOfficial && strings.IndexRune(remoteName, '/') == -1 {
        remoteName = "library/" + remoteName
    }

    mirrors = s.mirrors
}
```

注意`remoteName`和`localName`的区别。我们在拉取镜像时会发现有的镜像没有命名空间,
其实它是有一个默认值`library`。`remote`就是带了命名空间的规范化镜像名称.

### Registry API 版本
```go
if len(mirrors) == 0 && (isOfficial || endpoint.Version == registry.APIVersion2) {
    j := job.Eng.Job("trust_update_base")
    if err = j.Run(); err != nil {
        return job.Errorf("error updating trust base graph: %s", err)
    }

    if err := s.pullV2Repository(job.Eng, r, job.Stdout, localName, remoteName, tag, sf, job.GetenvBool("parallel")); err == nil {
        if err = job.Eng.Job("log", "pull", logName, "").Run(); err != nil {
            log.Errorf("Error logging event 'pull' for %s: %s", logName, err)
        }
        return engine.StatusOK
    } else if err != registry.ErrDoesNotExist {
        log.Errorf("Error from V2 registry: %s", err)
    }
}

if err = s.pullRepository(r, job.Stdout, localName, remoteName, tag, sf, job.GetenvBool("parallel"), mirrors); err != nil {
    return job.Error(err)
}
```

从代码中可以看到有两种拉取，分别对应于仓库的两个 API（V1 和 V2)版本。V2 是一种较新的架构，改
动较大,具体可见本文后面的链接。下面简要介绍一下官方仓库的拉取流程(示例):

![ ][3]

1. `docker client`从官方 Index("index.docker.io/v1")查询镜像("samalba/busybox")的
   位置
2. Index 回复:

    - `samalba/busybox`在`Registry A`上
    - `samalba/busybox`的校验码
    - token

3. `docker client` 连接`Registry A`表示自己要获取`samalba/busybox`

4. `Registry A` 询问`Index` 这个客户端(`token/user`)是否有权限下载镜像
5. `Index`回复是否可以下载
6. 下载镜像的所有`layers`

V1 和 V2 的区分即是再`registry`这一层。我们使用私有仓库的时候，都要在拉取的时候指定
仓库的 URL,这时候的流程与官方相比就少了`Index`服务这一层，所以安全性就不高。而且
因为没有了校验码，即使镜像损坏，也无法检测到。V2 的设计就是想统一各个仓库之间的不
一致,规范其安全性和可靠性等。

具体再使用上,两者的区分现在主要是在官方镜像和非官方镜像之间(`isOfficial`参数),如果是官方镜像
(`ubuntun`,`library/ubuntu`),则是从`V2`拉取,否则是从`v1`拉取。下面就以 V2 为中心分
析拉取流程。

在拉取之前,有一个叫`trust_update_base`的`job`先执行了，从名字上便知是与安全相关
的，也是 V2 引入的安全机制之一。`docker`为此创建了一个新的
[libtrust](https://github.com/docker/libtrust)项目,感兴趣的可以自行参考一下。
`trust_update_base`对应的`job`在`github.com/docker/docker/trust`包中,不再详述。


### Tag 处理
镜像拉取首先要确定的是`tag`,分两种情况,一种是指定了`--all-tags`，需要获取所有
`tag`的信息,另外就是单个的`tag`，不管是自己指定的还是默认的`latest`。

`github.com/docker/docker/graph/pull.go#pullV2Repository`:
```go
tags, err := r.GetV2RemoteTags(remoteName, nil)
if err != nil {
    return err
}
for _, t := range tags {
    if downloaded, err := s.pullV2Tag(eng, r, out, localName, remoteName, t, sf, parallel); err != nil {
        return err
    } else if downloaded {
        layersDownloaded = true
    }
}
```

上面代码展示的便是要拉取所有`tag`的情况，用的仍是标准的`REST
API`,`GetV2RemoteTags`便是一个标准的 go 语言的`GET`方法实现，不在详述。
我们可以直接从日志中查看到相关信息(`docker pull -a centos`):

![ ][4]

也可以自己再命令行下用`curl`或`httpie`工具直接获取:

![ ][5]

返回的结果就是一个`tags`的`string`列表。有了`tags`列表，下面要做的就是遍历列表一
个一个地下载。

### Manifest
要下载一个镜像，我们要先知道它的一些关键信息，比如校验码，层级,各层的联系以及其
他各种细节。所以首先要下载的便是这些`manifest`数据:

![ ][6]

先看下`MainfestData`的定义:

```go
type ManifestData struct {
    Name          string             `json:"name"`
    Tag           string             `json:"tag"`
    Architecture  string             `json:"architecture"`
    FSLayers      []*FSLayer         `json:"fsLayers"`
    History       []*ManifestHistory `json:"history"`
    SchemaVersion int                `json:"schemaVersion"`
}
```

在命令行下之下用`httpie`获取`centos:5`的`manifest`信息如下:

(1). fslayers

各层的`checksum`信息

![ ][7]

(2). History

各层的详细信息,`json`格式:

![ ][8]

(3). 签名及其他

![ ][9]


`github.com/docker/docker/graph/pull.go#pullV2Tag`:

```go
log.Debugf("Pulling tag from V2 registry: %q", tag)
manifestBytes, err := r.GetV2ImageManifest(remoteName, tag, nil)
// 验证各项信息是否正确
manifest, verified, err := s.verifyManifest(eng, manifestBytes)

downloads := make([]downloadInfo, len(manifest.FSLayers))
```



```go
type downloadInfo struct {
    imgJSON    []byte
    img        *image.Image
    tmpFile    *os.File
    length     int64
    downloaded bool
    err        chan error
}
```


`downloadInfo`里最主要的信息便是镜像的`json`描述文件,也是从`ManifestData`中获取
的。`tmpFile`的类型为`*os.File`,表明真正的下载要开始了。

```go
for i := len(manifest.FSLayers) - 1; i >= 0; i-- {
    var (
        sumStr  = manifest.FSLayers[i].BlobSum
        imgJSON = []byte(manifest.History[i].V1Compatibility)
    )

    img, err := image.NewImgJSON(imgJSON)
    if err != nil {
        return false, fmt.Errorf("failed to parse json: %s", err)
    }
    downloads[i].img = img
```


### 下载

整个下载流程大概分为以下几步:

(1). 确认此镜像(`layer`)的`ID`是否已经存在:

```go
if s.graph.Exists(img.ID) {
    log.Debugf("Image already exists: %s", img.ID)
    continue
}
```

如果已经存在，表示本地已有此镜像，跳过。

(2). 冲突检测
之前提到过在镜像拉取之前有冲突检测，那个是针对指定的镜像名的(比如`ubuntu`),而此
处的冲突主要是针对指定镜像名的各个`layer`之间的。我们知道很多镜像底层的`layer`都
是共享的，所以如果我们同时在拉取两个不同的镜像，其各自的`layers`可能会有重叠的部
分，所以在每层`layer`拉取之前都要检测:

```go
if c, err := s.poolAdd("pull", "img:"+img.ID); err != nil {
    if c != nil {
        out.Write(sf.FormatProgress(utils.TruncateID(img.ID), "Layer already being pulled by another client. Waiting.", nil))
        <-c
        out.Write(sf.FormatProgress(utils.TruncateID(img.ID), "Download complete", nil))
    } else {
        log.Debugf("Image (id: %s) pull is already running, skipping: %v", img.ID, err)
    }
} 
```

(3). 下载
```go
// 本地文件名
tmpFile, err := ioutil.TempFile("", "GetV2ImageBlob")
if err != nil {
    return err
}

// 下载
r, l, err := r.GetV2ImageBlobReader(remoteName, sumType, checksum, nil)
if err != nil {
    return err
}
defer r.Close()
io.Copy(tmpFile, utils.ProgressReader(r, int(l), out, sf, false, utils.TruncateID(img.ID), "Downloading"))
```


`docker`有自己的临时目录，一般是`/var/lib/docker/tmp/`，里面都是这种
`GetV2RemoteTags`开头的文件。`GetV2ImageBlobReader`仍是执行`GET`请求,我们可以从
日志里看到其`URL`的格式:

![ ][10]

下载地址主要是由`Manifest`中的`blobSum`字段组成，在浏览器里粘贴就可以下载这个文
件（最终重定向到 AWS)。

在之前的`CmdPull` `job`中,有一个`parallel`的参数,它用来控制下载时各`layer`之间是否是并行下载:

```go
if parallel {
    downloads[i].err = make(chan error)
    go func(di *downloadInfo) {
        di.err <- downloadFunc(di)
    }(&downloads[i])
} else {
    err := downloadFunc(&downloads[i])
    if err != nil {
        return false, err
    }
}
```

`docker`版本`1.3`以上的都是并行下载。


## 镜像存储
在`docker`的使用过程中,本地缓存的镜像会越来越多，我们需要有一个组件来管理这些镜
像及其之间的关系，在
[Daemon 启动流程](http://hangyan.github.io/docker/docker-source-code-part3-daemon-start/#graphdriver)
中我们提到的`graph`就是起这个作用。`daemon`启动时会创建一个`Graph`实例用来管理镜像,我们新下载的镜像也需要
向其“报到”，以纳入整个镜像关系网(树)中。

```go
if d.tmpFile != nil {
    err = s.graph.Register(d.img,
        utils.ProgressReader(d.tmpFile, int(d.length), out, sf, false, utils.TruncateID(d.img.ID), "Extracting"))
    if err != nil {
        return false, err
    }
}
```

所以我们需要先探究以下`Graph`的具体构造。

### Graph
```go
type Graph struct {
    Root    string
    idIndex *truncindex.TruncIndex
    driver  graphdriver.Driver
}
```

三个字段:根目录,索引,底层`driver`.`TruncIndex`的作用是使我们可以用长 ID 的前缀来检索镜像:

```go
type TruncIndex struct {
    sync.RWMutex
    trie *patricia.Trie
    ids  map[string]struct{}
}
```

`patricia`是一个基数树的`golang`实现。具体可参见[go-patricia](https://github.com/tchap/go-patricia).

Dirver 则定义了一组抽象的文件系统接口:

```go
type Driver interface {
    ProtoDriver
    // Diff produces an archive of the changes between the specified
    // layer and its parent layer which may be "".
    Diff(id, parent string) (archive.Archive, error)
    // Changes produces a list of changes between the specified layer
    // and its parent layer. If parent is "", then all changes will be ADD changes.
    Changes(id, parent string) ([]archive.Change, error)
    // ApplyDiff extracts the changeset from the given diff into the
    // layer with the specified id and parent, returning the size of the
    // new layer in bytes.
    ApplyDiff(id, parent string, diff archive.ArchiveReader) (bytes int64, err error)
    // DiffSize calculates the changes between the specified id
    // and its parent and returns the size in bytes of the changes
    // relative to its base filesystem directory.
    DiffSize(id, parent string) (bytes int64, err error)
}
```

`PhotoDriver`则定义了一个`driver`的基本功能集：
```go
type ProtoDriver interface {
    // String returns a string representation of this driver.
    String() string
    // Create creates a new, empty, filesystem layer with the
    // specified id and parent. Parent may be "".
    Create(id, parent string) error
    // Remove attempts to remove the filesystem layer with this id.
    Remove(id string) error
    // Get returns the mountpoint for the layered filesystem referred
    // to by this id. You can optionally specify a mountLabel or "".
    // Returns the absolute path to the mounted layered filesystem.
    Get(id, mountLabel string) (dir string, err error)
    // Put releases the system resources for the specified id,
    // e.g, unmounting layered filesystem.
    Put(id string)
    // Exists returns whether a filesystem layer with the specified
    // ID exists on this driver.
    Exists(id string) bool
    // Status returns a set of key-value pairs which give low
    // level diagnostic status about this driver.
    Status() [][2]string
    // Cleanup performs necessary tasks to release resources
    // held by the driver, e.g., unmounting all layered filesystems
    // known to this driver.
    Cleanup() error
}
```



在`daemon`启动时，调用了`NewGraph`接口，如果本身没有镜像的话，那么这个函数所做的
基本上只是一些变量的初始化,如果本身已经有镜像存在，则需要重新读取并建立它们之间
的联系。



### Register

一个新的镜像(layer)向`Graph`注册主要有以下流程:

#### 1. ID 验证

```go
if err := utils.ValidateID(img.ID); err != nil {
    return err
}
```

用正则表达式检验其是否包含非法字符,规则是只能包含英语字母和数字

#### 2. 检查是否已经存在

```go
if graph.Exists(img.ID) {
    return fmt.Errorf("Image %s already exists", img.ID)
}

if err := os.RemoveAll(graph.ImageRoot(img.ID)); err != nil && !os.IsNotExist(err) {
    return err
}

graph.driver.Remove(img.ID)
```

之所以要这么多步删除是为了应对一些特殊情况，比如切换`graph driver`等，这时候就可
能信息不一致的地方。

#### 3. 创建`image rootfs`
到这一步就牵扯到了具体的`graph driver`实现了，`ubuntu`上现在都是`aufs`,下面就以
`aufs`为例探讨。

```go
if err := graph.driver.Create(img.ID, img.Parent); err != nil {
	return fmt.Errorf("Driver %s failed to create image rootfs %s: %s", graph.driver, img.ID, err)
}
```

首先，创建`mnt`和`diff`两个目录(在`/var/lib/docker/aufs`下):

`github.com/docker/docker/daemon/graphdriver/aufs/aufs.go#Create`:
```go
if err := a.createDirsFor(id); err != nil {
	return err
}
```

`github.com/docker/docker/daemon/graphdriver/aufs/aufs.go#createDirsFor`:
```go
func (a *Driver) createDirsFor(id string) error {
    paths := []string{
        "mnt",
        "diff",
    }

    for _, p := range paths {
        if err := os.MkdirAll(path.Join(a.rootPath(), p, id), 0755); err != nil {
            return err
        }
    }
    return nil
}
```

然后，创建`layers`目录，里面记录了镜像之间的层级关系:

`github.com/docker/docker/daemon/graphdriver/aufs/aufs.go#Create`:
```go
f, err := os.Create(path.Join(a.rootPath(), "layers", id))
if err != nil {
    return err
}

if parent != "" {
    ids, err := getParentIds(a.rootPath(), parent)
    if err != nil {
        return err
    }

    if _, err := fmt.Fprintln(f, parent); err != nil {
        return err
    }
    for _, i := range ids {
        if _, err := fmt.Fprintln(f, i); err != nil {
            return err
        }
    }
}
```

我们可以在`/var/lib/docker/aufs/layers`目录下找一个文件看一下其中的内容:
![ ][11]

每一行代表一个镜像，每一行都是上一行的`parent`镜像。可以猜想大多数的最后一行都一
样，即来自于同一个基本镜像
`511136ea3c5a64f264b78b5433614aec563103b4d4702f3ba7d4d2698e22c158`。

#### 4. 解压数据,计算 layersize
`github.com/docker/docker/graph/graph.go#Register`:
```go
img.SetGraph(graph)
if err := image.StoreImage(img, layerData, tmp); err != nil {
	return err
}
```

`tmp`即是`/var/lib/docker/graph/_tmp`，用来暂时存储数据。

首先，解压数据，并存入`/var/lib/docker/diff`中:

`github.com/docker/docker/image/image.go#StoreImage`:
```go
layerDataDecompressed, err := archive.DecompressStream(layerData)

if layerTarSum, err = tarsum.NewTarSum(layerDataDecompressed, true, tarsum.VersionDev); err != nil {
    return err
}

if size, err = driver.ApplyDiff(img.ID, img.Parent, layerTarSum); err != nil {
    return err
}
```

实际的数据解压存储实在`driver.ApplyDiff`中执行的,`aufs`的实现中是不需要`parent
image id`的，所以较为简单,只用简单地解压数据并计算大小即可:


`github.com/docker/daemon/graphdriver/aufs/aufs.go`:

```go
func (a *Driver) ApplyDiff(id, parent string, diff archive.ArchiveReader) (bytes int64, err error) {
    // AUFS doesn't need the parent id to apply the diff.
    if err = a.applyDiff(id, diff); err != nil {
        return
    }

    return a.DiffSize(id, parent)
}
```

```go
// 解压数据
func (a *Driver) applyDiff(id string, diff archive.ArchiveReader) error {
    return chrootarchive.Untar(diff, path.Join(a.rootPath(), "diff", id), nil)
}
```

```go
// 遍历目录计算大小
func (a *Driver) DiffSize(id, parent string) (bytes int64, err error) {
    // AUFS doesn't need the parent layer to calculate the diff size.
    return utils.TreeSize(path.Join(a.rootPath(), "diff", id))
}
```


计算好的大小会暂时存在`/var/lib/docker/graph/_tmp/<ID>/layersize`文件中:

`github.com/docker/docker/image/image.go#StoreImage`:

```go
img.Size = size
if err := img.SaveSize(root); err != nil {
    return err
}
```

同样的,`json`描述文件也会暂时存在`/var/lib/docker/graph/_tmp/<ID>/json`中:

```go
f, err := os.OpenFile(jsonPath(root), os.O_CREATE|os.O_WRONLY|os.O_TRUNC, os.FileMode(0600))
if err != nil {
    return err
}

defer f.Close()

return json.NewEncoder(f).Encode(img)
```


在整个流程中也包含了校验码的对比验证:

`github.com/docker/docker/image/image.go#StoreImage`:
```go
checksum := layerTarSum.Sum(nil)

if img.Checksum != "" && img.Checksum != checksum {
    log.Warnf("image layer checksum mismatch: computed %q, expected %q", checksum, img.Checksum)
}
```

`image.StoreImage`结束后，将`_tmp`下的数据移到正式的目录里面:

`github.com/docker/docker/graph/graph.go#Register`:
```go
if err := os.Rename(tmp, graph.ImageRoot(img.ID)); err != nil {
    return err
}
```

#### 5. 将镜像 ID 加入索引

`github.com/docker/docker/graph/graph.go#Register`:
```go
graph.idIndex.Add(img.ID)
```

即前面所说的`TruncIndex`中。


### TagStore
`Graph`结构存储了各个镜像的元数据及其之间的关系，但仍有一个维度的数据它没有建立
关联：名字。我们需要一个能够将镜像名字和`tag`(`ubuntu：14.04`)与其镜像
ID(`d0955f21bf24`)关联起来的数据结构，这就是`TagStore`：

`github.com/docker/docker/graph/tags.go`
```go
type TagStore struct {
    path               string
    graph              *Graph
    mirrors            []string
    insecureRegistries []string
    Repositories       map[string]Repository
    sync.Mutex
    // FIXME: move push/pull-related fields
    // to a helper type
    pullingPool map[string]chan struct{}
    pushingPool map[string]chan struct{}
}

type Repository map[string]string

```

在[Daemon 启动流程中](http://hangyan.github.io/docker/docker-source-code-part3-daemon-start/)
中已经提到,`TagStore`的数据存在`/var/lib/docker/graph/repositories-<driver-name>`中，对`aufs`来说，就是
`repositories-aufs`,里面记录了镜像名与镜像 ID 之间的映射：

![ ][12]


在镜像向`Graph`注册之后，我们也需要向`TagStore`注册:

`github.com/docker/docker/graph/pull.go#pullV2Tag`:

```go
if err = s.Set(localName, tag, downloads[0].img.ID, true); err != nil {
    return false, err
}
```


`localname`是没有进行过命名空间规整的镜像名（即不会有额外的`library/`）。
`downloadds[0].img.ID`即是与镜像名相关联的镜像 ID（在`Manifest`数据中的
`History`列表中的第一位）。

首先获取这个镜像的`Image`对象:

`github.com/docker/docker/graph/tags.go#Set`:
```go
img, err := store.LookupImage(imageName)
store.Lock()
defer store.Unlock()
if err != nil {
    return err
}
```

如果在`TagStore`中找不到的话会到`Graph`中寻找:
`github.com/docker/docker/graph/tags.go#LookupImage`:
```go
img, err := store.GetImage(repos, tag)
store.Lock()
defer store.Unlock()
if err != nil {
    return nil, err
} else if img == nil {
    if img, err = store.graph.Get(name); err != nil {
        return nil, err
    }
}
return img,nil
```

之后便是对`repoName`和`tag`的校验,最后将相关信息写入`TagStore`的
`Repositories`(`map`)中。

```go
if err := store.reload(); err != nil {
    return err
}
var repo Repository
if r, exists := store.Repositories[repoName]; exists {
    repo = r
    if old, exists := store.Repositories[repoName][tag]; exists && !force {
        return fmt.Errorf("Conflict: Tag %s is already set to image %s, if you want to replace it, please use -f option", tag, old)
    }
} else {
    repo = make(map[string]string)
    store.Repositories[repoName] = repo
}
repo[tag] = img.ID
return store.save()
```


## 总结
镜像的拉取和存储主要与`/var/lib/docker/`下面的四个目录有关:

1. ./aufs 镜像实际数据,layer 关系
2. ./graph 镜像 json 描述文件,layersize
3. ./tmp 镜像临时下载目录
4. ./trust 认证相关

所以能实际结合这几个目录下的数据来分析源码,一定会事半功倍.



[1]: http://hangyan.github.io/images/posts/docker/source-5/server-handlers.png "server-handlers"
[2]: http://hangyan.github.io/images/posts/docker/source-5/image-handlers.png "image-handlers"
[3]: http://hangyan.github.io/images/posts/docker/source-5/docker_pull_chart.png "docker_pull_chart"
[4]: http://hangyan.github.io/images/posts/docker/source-5/tags.png "tags"
[5]: http://hangyan.github.io/images/posts/docker/source-5/tags-http.png "tags-http"
[6]: http://hangyan.github.io/images/posts/docker/source-5/centos-5-manifest.png "centos-5-manifest"
[7]: http://hangyan.github.io/images/posts/docker/source-5/fslayers.png "fslayers"
[8]: http://hangyan.github.io/images/posts/docker/source-5/history.png "history"
[9]: http://hangyan.github.io/images/posts/docker/source-5/sigs.png "sigs"
[10]: http://hangyan.github.io/images/posts/docker/source-5/pulling-blob.png "pulling-blob"
[11]: http://hangyan.github.io/images/posts/docker/source-5/layers.png "layers"
[12]: http://hangyan.github.io/images/posts/docker/source-3/repos.png "repos"
[13]: http://hangyan.github.io/images/posts/docker/source-5/commands.png "commands"

## 参考链接
1. [V2 Registry Talk](https://news.ycombinator.com/item?id=8789775)
2. [Registry next generation](https://github.com/docker/docker-registry/issues/612)
3. [Proposal: JSON Registry API V2.1](https://github.com/docker/docker/issues/9015)
4. [The Docker Hub and the Registry spec](https://docs.docker.com/reference/api/hub_registry_spec/)
5. [Proposal: Private registry name-spacing as part of V2 image names](https://github.com/docker/docker/issues/9076)
6. [Proposal: Self-describing images](https://github.com/docker/docker/issues/6805)
7. [Proposal: Provenance step 1 - Transform images for validation and verification](https://github.com/docker/docker/issues/8093)
