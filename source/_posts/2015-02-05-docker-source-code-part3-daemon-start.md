---
layout: post
title: docker 源码分析(3) -- daemon 启动流程
tags: [docker]
imagefeature:
comments: true
share: true
excerpt: "从主函数解析 daemon 进程的启动流程"
---


本篇的主要内容是关于`docker daemon`的启动流程。其主要内容均包含在
`github.com/docker/docker/docker/daemon.go`文件中的`mainDaemon`函数中，本文即按
其执行流程分析源码。因为所涉源码较多，所以所涉部分多是点到为止，详细分析会在后续
分专篇讲述。
<!--more-->

# Engine

在`mainDaemon`的开始处，在确认参数解析无误后,首先便生成了一个`Engine`的实例:

```go
eng := engine.New()
signal.Trap(eng.Shutdown)
```

`Engine`可以说是`docker`的核心。它用来执行`docker`的各种操作(统一为`job`的形式)，
管理`container`的存储。其结构定义为 ：

```go
type Engine struct {
	handlers   map[string]Handler
	catchall   Handler
	hack       Hack // data for temporary hackery (see hack.go)
	id         string
	Stdout     io.Writer
	Stderr     io.Writer
	Stdin      io.Reader
	Logging    bool
	tasks      sync.WaitGroup
	l          sync.RWMutex // lock for shutdown
	shutdown   bool
	onShutdown []func() // shutdown handlers
}
```

大部分均可见名知意，下面对部分字段进行详细解析。

## Handler

`Engine`结构体中中最关键的便是`handlers`映射表，`docker daemon`启动时会向其中注册各种功
能的`handler`，比如关于网络设置的、`web server`、版本等等，然后就可以通过名字调用进行
初始化:

```go
type Handler func(*Job) Status
```

各个模块在初始化时只要设置好相应环境变量并注册一个`job`即可。统一的函数接口能够
让`docker`内部各组件在代码结构和执行流程上更加清晰一致。

## Job
`job`是`Engine`最为基本的执行单元。所有的`docker`操作，比如启动一个
`container`,在`container`内部执行一个程序，从网络`pull`一个镜像等等，都可以用
`job`来表示。

```go
type Job struct {
	Eng     *Engine
	Name    string
	Args    []string
	env     *Env
	Stdout  *Output
	Stderr  *Output
	Stdin   *Input
	handler Handler
	status  Status
	end     time.Time
	closeIO bool
}
```


```go
const (
	StatusOK       Status = 0
	StatusErr      Status = 1
	StatusNotFound Status = 127
)
```


从结构体的定义来看，`job`与`unix`上的进程的结构表示非常类似：名字、参数、环境变
量、标准输入输出、退出状态(0 表示成功，其他表示错误)……我们完全可以将其当作像进程一样的概念
来看待。

## Initializes

`New`函数用来初始化一个`Engine`,基本上只是对各变量进行简单的初始化:

```go
func New() *Engine {
    eng := &Engine{
        handlers: make(map[string]Handler),
        id:       utils.RandomString(),
        Stdout:   os.Stdout,
        Stderr:   os.Stderr,
        Stdin:    os.Stdin,
        Logging:  true,
    }
    eng.Register("commands", func(job *Job) Status {
        for _, name := range eng.commands() {
            job.Printf("%s\n", name)
        }
        return StatusOK
    })
    // Copy existing global handlers
    for k, v := range globalHandlers {
        eng.handlers[k] = v
    }
    return eng
}
```

注意点:

1. `Engine id`是一个随机字符串
2. 注册了一个`commands`的`handler`，用来返回`Engine`所支持的`commands`(`handlers`表中的`key`)列表。
3. 如果已经有预定义好的`globalHandlers`，也添加到`Engine`的`handlers`表中.


## Shutdown

`Engine`关闭的流程大概如下:

1. 不再接受新的执行`job`的请求
2. 等待所有正在执行中的`job`结束
3. 并发调用已经注册的各个`shutdown handlers`
4. 所有`handlers`结束或者等待 15 秒后返回

具体可参考`github.com/docker/docker/engine/engine.go#Shutdown()`。


上面`mainDaemon`中`Trap`的设置可以让`Engine`像大多数`unix`程序一样在接收到信号时做
一些指定的操作:

- `SIGINT` 或者 `SIGTERM`, 直接调用`eng.Shutdown`,然后程序结束
- `SIGINT` 或者 `SIGTERM` 在`eng.Shutdown`执行完成之前重复了 3 次,那么就直接停止执行并且直接结束程序
- 如果`DEBUG`环境变量被设置，`SIGQUIT`会直接让程序退出而不调用`eng.Shutdown`

# Builtins

```go
if err := builtins.Register(eng); err != nil {
    log.Fatal(err)
}
```


`builtins`包主要用来给`Engine`注册一些内部使用的`handlers` : 网络设置、
`apiserver`、`Events`设置和`version`信息 :

```go
func Register(eng *engine.Engine) error {
    if err := daemon(eng); err != nil {
        return err
    }
    if err := remote(eng); err != nil {
        return err
    }
    if err := events.New().Install(eng); err != nil {
        return err
    }
    if err := eng.Register("version", dockerVersion); err != nil {
        return err
    }

    return nil
}
```


因为只是注册,具体的执行还在后面，所以这里暂时不深入探讨各个`handler`的详细内容，
等分析到实际执行的时候再结合运行时信息详细探讨，理解起来应该更容易一些。这里只列
出注册的`handlers`映射信息:

Name | Handler
---- | -------
init_networkdriver |  bridge.InitDriver
serveapi           |  apiserver.ServeApi
acceptconnections  |  apiserver.AcceptConnections
version            |  dockerVersion

## Version
因为`dockerVersion`的实现比较简单，所以就直接写在了
`github.com/docker/docker/builtins/builtins.go`里面:

```go
func dockerVersion(job *engine.Job) engine.Status {
    v := &engine.Env{}
    v.SetJson("Version", dockerversion.VERSION)
    v.SetJson("ApiVersion", api.APIVERSION)
    v.SetJson("GitCommit", dockerversion.GITCOMMIT)
    v.Set("GoVersion", runtime.Version())
    v.Set("Os", runtime.GOOS)
    v.Set("Arch", runtime.GOARCH)
    if kernelVersion, err := kernel.GetKernelVersion(); err == nil {
        v.Set("KernelVersion", kernelVersion.String())
    }
    if _, err := v.WriteTo(job.Stdout); err != nil {
        return job.Error(err)
    }
    return engine.StatusOK
}
```

我们可以直接通过执行`docker version`命令来查看其大概效果:

![ ][1]

## Events

我们可以先通过`docker events`命令来看看`docker`中`Events`是干嘛用的。如图，启动一个
`container`：

![ ][2]

在另一个窗口的`docker events`命令显示结果:
![ ][3]

可见`Events`是类似于 log 的一种东西，不过是一种结构化的记录方式，而且只记录特定的
运行时信息。

```go
const eventsLimit = 64

type listener chan<- *utils.JSONMessage

type Events struct {
    mu          sync.RWMutex
    events      []*utils.JSONMessage
    subscribers []listener
}

func New() *Events {
    return &Events{
        events: make([]*utils.JSONMessage, 0, eventsLimit),
    }
}
```

而前面提到的`events.New().Install(eng)`也是向`Engine`注册了一些`handlers`:

```go
func (e *Events) Install(eng *engine.Engine) error {
    // Here you should describe public interface
    jobs := map[string]engine.Handler{
        "events":            e.Get,
        "log":               e.Log,
        "subscribers_count": e.SubscribersCount,
    }
    for name, job := range jobs {
        if err := eng.Register(name, job); err != nil {
            return err
        }
    }
    return nil
}
```

具体的函数实现则不再赘述。


# Registry
```go
if err := registry.NewService(daemonCfg.InsecureRegistries).Install(eng); err != nil {
    log.Fatal(err)
}
```

`registry`主要是给`Engine`提供认证和搜索官方(`dockerhub`)镜像的能力：

```go
func (s *Service) Install(eng *engine.Engine) error {
    eng.Register("auth", s.Auth)
    eng.Register("search", s.Search)
    return nil
}
```


如代码所示，`Registry`注册了`auth`和`search`两个`handler`。


# Daemon

经过前面的那么多设置,`Engine`算是配置的差不多了，下面就是对`daemon`进行各项配置
的时候了:

```go
go func() {
    d, err := daemon.NewDaemon(daemonCfg, eng)
    if err != nil {
        log.Fatal(err)
    }
    
    log.Infof("docker daemon: %s %s; execdriver: %s; graphdriver: %s",
        dockerversion.VERSION,
        dockerversion.GITCOMMIT,
        d.ExecutionDriver().Name(),
        d.GraphDriver().String(),
    )
    
    if err := d.Install(eng); err != nil {
        log.Fatal(err)
    }
    
    b := &builder.BuilderJob{eng, d}
    b.Install()
    
    if err := eng.Job("acceptconnections").Run(); err != nil {
        log.Fatal(err)
    }
}()
```

主要内容如下:

1. `daemon`各个模块的设置，创建`daemon`。这部分内容非常长，下面将详述。
2. 打印一些关键日志信息。如下图所示:

    ![ ][20]

3.  向`Engine`注册`daemon`所提供的各种`handlers`,主要就是`docker client`各种命令
    的后台实现:

    ![ ][21]

    ![ ][22]

4. `docker build`的后台`handler`实现。因为这个命令实现比较复杂，所以单列。
5. 在`daemon`设置完成后即启动`api server`准备接受请求。

## Config

`Config`定义了`docker daemon`的各项配置:

```go
type Config struct {
	Pidfile                     string
	Root                        string
	AutoRestart                 bool
	Dns                         []string
	DnsSearch                   []string
	Mirrors                     []string
	EnableIptables              bool
	EnableIpForward             bool
	EnableIpMasq                bool
	DefaultIp                   net.IP
	BridgeIface                 string
	BridgeIP                    string
	FixedCIDR                   string
	InsecureRegistries          []string
	InterContainerCommunication bool
	GraphDriver                 string
	GraphOptions                []string
	ExecDriver                  string
	Mtu                         int
	DisableNetwork              bool
	EnableSelinuxSupport        bool
	Context                     map[string][]string
	TrustKeyPath                string
	Labels                      []string
}
```


```go
type Daemon struct {
	ID             string
	repository     string
	sysInitPath    string
	containers     *contStore
	execCommands   *execStore
	graph          *graph.Graph
	repositories   *graph.TagStore
	idIndex        *truncindex.TruncIndex
	sysInfo        *sysinfo.SysInfo
	volumes        *volumes.Repository
	eng            *engine.Engine
	config         *Config
	containerGraph *graphdb.Database
	driver         graphdriver.Driver
	execDriver     execdriver.Driver
	trustStore     *trust.TrustStore
}
```



从这些配置项也可以看出，很多都是与`docker`启动时的参数一一对应的。`NewDaemon`函
数即通过这些参数来进行`daemon`的各项设置:

## Settings
从上面`Config`和`Daemon`的定义也可以看出，二者包含了`docker`运行时需要关注
的绝大部分内容及组件。而具体的设置由
`github.com/docker/docker/daemon/daemon.go#NewDaemonFromDirectory`完成，因为比较
琐碎，所以将其归为以下几类介绍:

### network args

因为网络参数比较多，有的还有冲突，所有还要进行一定的检测。关于网络的设置主要由以
下几项:

1. `MTU`,容器网络的最大传输单元。未指定则使用默认值: 1500。如果网络环境的自定义程
   度较高，则`MTU`需要小心设置，不然可能因为额外的封包解包过程导致包大小超过`MTU`而
   被丢弃。
2. `--bridge` 和 `--bip` 参数不能同时指定。因为`bridge`是用来创建自定义的
   `bridge`网络，而`--bip`是用来给默认的`docker0`指定其他地址和掩码的。
3. `--iptables=false` 和 `--icc=false`不能同时指定。因为`ICC`依赖于`iptables`

    ...

### system

- `pidfile`的创建和管理

```go
if config.Pidfile != "" {
  if err := utils.CreatePidFile(config.Pidfile); err != nil {
    return nil, err
  }
  eng.OnShutdown(func() {
    // Always release the pidfile last, just in case
    utils.RemovePidFile(config.Pidfile)
  })
}
```

   使用`pid`文件可以说是`linux`上大多数`daemon`服务的一种通用模式了: 没有则创建，
   并且在程序退出时删除(通过`shutdown handler`来处理)。

- 操作系统及内核版本检测,要求 linux 3.8 以上的 kernel.

```go
if runtime.GOOS != "linux" {
  return nil, fmt.Errorf("The Docker daemon is only supported on linux")
}

if err := checkKernelAndArch(); err != nil {
  return nil, err
}
```

-  权限检测,`docker`需要`root`权限运行

```go
if os.Geteuid() != 0 {
  return nil, fmt.Errorf("The Docker daemon needs to be run as root")
}
```


-  `TempDir`设置

这里的`TempDir`是相对于`docker`的目录而言的,并不是指系统的`/tmp`目录。从参数设置
上可以看到默认的根目录为`/var/lib/docker`:
```go
flag.StringVar(&config.Root, []string{"g", "-graph"}, "/var/lib/docker", "Path to use as the root of the Docker runtime")
```

如果使用默认值，则`TempDir`为`/var/lib/docker/tmp`：
```go
func TempDir(rootDir string) (string, error) {
    var tmpDir string
    if tmpDir = os.Getenv("DOCKER_TMPDIR"); tmpDir == "" {
        tmpDir = filepath.Join(rootDir, "tmp")
    }
    err := os.MkdirAll(tmpDir, 0700)
    return tmpDir, err
}
```



- SELinux

检测是否开启`SELinux`支持。`SELinux`和`Apparmor`是 docker 支持的两种安全机制，`SELinux`功能强大，架构也比较复
杂，`AppArmor`则相反。

```go
if !config.EnableSelinuxSupport {
	selinuxSetDisabled()
}
```

- Docker root directory

`Docker`所有文件存储的根目录，默认为`/var/lib/docker`。
    

### graphdriver
`graph driver`是主要用来管理容器文件系统及镜像存储的组件,与宿主机对各文件系统的支持
相关。比如`ubuntu`上默认使用的是`AUFS`,`Centos`上是`devicemapper`,`Coreos`上则是`btrfs`。
`graph driver`定义了一个统一的、抽象的接口,以一种可扩展的方式对各文件系统提供了支持。

```go

// Set the default driver
graphdriver.DefaultDriver = config.GraphDriver

// Load storage driver
driver, err := graphdriver.New(config.Root, config.GraphOptions)
if err != nil {
  return nil, err
}
log.Debugf("Using graph driver %s", driver)

```

因为`config.GraphDriver`并没有设置(没有供用户指定的参数选项)，所以`graphDriver`会从其支持的文件系统列表中
一个一个检测系统是否支持,找到一个支持的即设为要用的 driver :

```go
for _, name := range priority {
  driver, err = GetDriver(name, root, options)
  if err != nil {
    if err == ErrNotSupported || err == ErrPrerequisites || err == ErrIncompatibleFS {
        continue
	}
    return nil, err
  }
  return driver, nil
}
```

`priority`列表为:

```go
priority = []string{
	"aufs",
	"btrfs",
	"devicemapper",
	"vfs",
	// experimental, has to be enabled manually for now
	"overlay",
}
```

如果使用的是`btrfs`,因为其与`SELinux`的不兼容，所以还要进行一些检测:

```go
if selinuxEnabled() && config.EnableSelinuxSupport && driver.String() == "btrfs" {
    return nil, fmt.Errorf("SELinux is not supported with the BTRFS graph driver!")
}
```

之后检测`/var/lib/docker/containers`目录是否存在，不存在则创建。我们来看看
`containers`目录下的内容：

![ ][23]

每个`container`创建的时候，与网络有关的配置文件
(`/etc/hosts`,`/etc/resolv.conf`等)与其他文件的处理是不同的,他们是通过挂载的方式
供`container`使用的，有点类似于`docker container`本身的存储方式: 一个只读的层，
加上一些可写的层。`containers`目录就是用来存储这些信息的。



### graph
```go
g, err := graph.NewGraph(path.Join(config.Root, "graph"), driver)
if err != nil {
  return nil, err
}
```


`Graph`是用来存储标记的文件系统镜像以及他们之间的关系的组件:

```go
type Graph struct {
	Root    string
	idIndex *truncindex.TruncIndex
	driver  graphdriver.Driver
}
```

其中`idIndex`的作用是使我们可以使用长 id 的前缀来检索镜像,`Root`为`Graph`的根目录，一般为`/var/lib/docker/graph`。`NewGraph`即是用此目
录下的文件来重建镜像索引。我们可以查看此目录下的目录的结构:

![ ][4]

每一个镜像一个目录，下面包含一个描述镜像信息的 json 文件，也包含了记录镜像大小的
`layersize`文件。我们用一些实例来对比查看一下,下图是`docker images --tree`的部分
结果:

![ ][5]

我们选取`487e08`镜像来对照,json 文件记录了其`parent image`的 id、创建时间、大小等
等信息。这个大小与 layersize 文件中的相一致。

![ ][6]

![ ][7]

### volumes

`Volumes`是一种特殊的目录，其数据可以被一个或多个`container`共享,它和创建它的
`container`的生命周期分离开来，在`container`被删去之后能继续存在。在实现上，使用
的依然是只读层和读写层结合(`union file system`)的方式。

```go
volumesDriver, err := graphdriver.GetDriver("vfs", config.Root, config.GraphOptions)
if err != nil {
    return nil, err
}

volumes, err := volumes.NewRepository(path.Join(config.Root, "volumes"), volumesDriver)
if err != nil {
    return nil, err
}
```

`VFS`是一个中间层,下面是个各种文件系统实现，对外提供的则是统一的访问接口,这非常
类似我们之前提到的`GraphDriver`的机制。刚开始看这段代码很难知道它是干嘛用的，但我们还可以
仿照之前`Graph`部分先对`/var/lib/docker/volumes`目录进行一番探究。

我们先用官方的例子创建一个包含`Volumes`的`container`：

![ ][8]

然后通过`docker inspect`查看与`Volumes`相关的信息:

![ ][9]

到获取到的目录去看下:

![ ][10]

里面什么也没有。我们进到 container 内部在`/webapp`目录下创建一个文件看看:

![ ][11]

可以确定，`/var/lib/docker/vfs`目录下的目录是用来存储`Volumes`中实际数据的。我们
再来看看`/var/lib/docker/volumes`目录下的内容:

![ ][12]

可以看到，这个目录只用来存储关于`Volumes`的关键信息的。


明白了这些之后，就会发现上面的代码和之前的与`Graph`有关的代码是非常类似的 : 初始
化`driver`，然后从相应目录里读取原有的关于`image`或者`container`的信息并加载。


### repository
```go
repositories, err := graph.NewTagStore(path.Join(config.Root, "repositories-"+driver.String()), g, config.Mirrors, config.InsecureRegistries)
if err != nil {
    return nil, fmt.Errorf("Couldn't create Tag store: %s", err)
}
```

我们依然先来查看下相关的文件: `/var/lib/docker/repositories-aufs` :

![ ][13]

整个 json 文件记录了所有的镜像的不同的 tag 及其对应的 id.从函数名`NewTagStore`上也可
以看出，`tag`信息的记录是其主要功能之一。

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
```

`pullingPool`记录有哪些镜像正在被下载，若某一个镜像正在被下载，则驳回其他`Docker
Client`发起下载该镜像的请求。`pushingPool`记录有哪些镜像正在被上传，若某一个镜像
正在被上传，则驳回其他`Docker Client`发起上传该镜像的请求；


### trust

```go
trustDir := path.Join(config.Root, "trust")
if err := os.MkdirAll(trustDir, 0700); err != nil && !os.IsExist(err) {
	return nil, err
}
t, err := trust.NewTrustStore(trustDir)
if err != nil {
	return nil, fmt.Errorf("could not create trust store: %s", err)
}
```

还是先看`/var/lib/docker/trust`下的内容:

![ ][14]

跟认证签名有关的一些信息。这个文件是从下面这个地方获取到的:

```go
var baseEndpoints = map[string]string{"official": "https://dvjy3tqbc323p.cloudfront.net/trust/official.json"}
```

然后用其中的内容来初始化`TrustStore`:

```go
type TrustStore struct {
	path          string
	caPool        *x509.CertPool
	graph         trustgraph.TrustGraph
	expiration    time.Time
	fetcher       *time.Timer
	fetchTime     time.Duration
	autofetch     bool
	httpClient    *http.Client
	baseEndpoints map[string]*url.URL

	sync.RWMutex
}
```


### init_networkdriver

```go
if !config.DisableNetwork {
    job := eng.Job("init_networkdriver")

    job.SetenvBool("EnableIptables", config.EnableIptables)
    job.SetenvBool("InterContainerCommunication", config.InterContainerCommunication)
    job.SetenvBool("EnableIpForward", config.EnableIpForward)
    job.SetenvBool("EnableIpMasq", config.EnableIpMasq)
    job.Setenv("BridgeIface", config.BridgeIface)
    job.Setenv("BridgeIP", config.BridgeIP)
    job.Setenv("FixedCIDR", config.FixedCIDR)
    job.Setenv("DefaultBindingIP", config.DefaultIp.String())

    if err := job.Run(); err != nil {
        return nil, err
    }
}
```

前面提到在`Builtins`里注册了这个`handler`，这里就利用启动参数进行了相关环境变
量的设置并真正开始启动这个 Job。主要内容如下:

1. `bridge`及其 ip 设置，一般都是使用默认的`docker0`。

![ ][15]

2. `iptables`及`ipforward`设置。

3. `fixed cidr` 设置。这个可以用来限制`contaienr`从`docker0`获取到的`ip`地址的范
   围。

4. 注册了一些供以后进行各个容器的网络设置的`handlers`:
```go
for name, f := range map[string]engine.Handler{
    "allocate_interface": Allocate,
    "release_interface":  Release,
    "allocate_port":      AllocatePort,
    "link":               LinkContainers,
} {
    if err := job.Eng.Register(name, f); err != nil {
        return job.Error(err)
    }
}

```



### linkgraph.db
```go
graphdbPath := path.Join(config.Root, "linkgraph.db")
graph, err := graphdb.NewSqliteConn(graphdbPath)
if err != nil {
    return nil, err
}

```

`/var/lib/docker/linkgraph.db`是一个 SQLITE3 的数据库文件。里面有两个表: `edge` 和
`entity`(两个图理论中常用的概念)。查看其内容:

![ ][24]

![ ][25]


`edge`里存储了容器的名字和 id,`entity`只存储了容器的 id。`daemon`通过这个数据库来重
建容器名称与 id 的关联。


### execdriver

```go
sysInfo := sysinfo.New(false)
ed, err := execdrivers.NewDriver(config.ExecDriver, config.Root, sysInitPath, sysInfo)
if err != nil {
	return nil, err
}
```

`docker`最开始使用的是`linux`的`lxc`作为其底层的容器执行引擎，后来自己开发了
`libcontainer`，用来替代`lxc`，所以我们现在看到`docker info`里显示的`Excution
Driver`是`native`:

![ ][16]

`sysinfo`是`cgroup`相关的一些系统信息，`lxc exec driver`初始化时需要从其中获取关于系统中`apparmor`的一些信息，但`native exec driver`不需要。

```go
type SysInfo struct {
	MemoryLimit            bool
	SwapLimit              bool
	IPv4ForwardingDisabled bool
	AppArmor               bool
}
```


```go
func NewDriver(name, root, initPath string, sysInfo *sysinfo.SysInfo) (execdriver.Driver, error) {
    switch name {
    case "lxc":
        return lxc.NewDriver(root, initPath, sysInfo.AppArmor)
    case "native":
        return native.NewDriver(path.Join(root, "execdriver", "native"), initPath)
    }
    return nil, fmt.Errorf("unknown exec driver %s", name)
}
```

看看代码中提到的目录`/var/lib/docker/execdriver/native`:

![ ][17]

又是一堆`container`或者镜像的`id`,既然是执行引擎了，多半是关于`container`的一些
运行时信息，挑一个进去查看一下:

![ ][18]

`state.json`主要描述了此`container`所在`cgroup`的相关目录，网络状态,以及主进程的
`pid`及启动时间。`container.json`包含信息较多，部分截图如下:

![ ][19]

1. 各个设备的访问权限,主要是`/dev`下面那些
2. 一些特殊文件的信息。比如`/etc/hosts`,`Volumes`,`/etc/resolv.conf`等等
3. 网络详细信息
4. capabilites
5. namespaces
6. 环境变量




## Restore
经过前面各个组件的设置及初始化，终于到了`daemon`的创建了：

```go
daemon := &Daemon{
    ID:             trustKey.PublicKey().KeyID(),
    repository:     daemonRepo,
    containers:     &contStore{s: make(map[string]*Container)},
    execCommands:   newExecStore(),
    graph:          g,
    repositories:   repositories,
    idIndex:        truncindex.NewTruncIndex([]string{}),
    sysInfo:        sysInfo,
    volumes:        volumes,
    config:         config,
    containerGraph: graph,
    driver:         driver,
    sysInitPath:    sysInitPath,
    execDriver:     ed,
    eng:            eng,
    trustStore:     t,
}
if err := daemon.restore(); err != nil {
    return nil, err
}
```

基本上用到了我们前面设置好的各个组件。之后的`restore`便开始加载原有的`container`，
将设为自启动的`container`启动。


## Shutdown

前面提到过`Engine`在关闭时会调用各个注册好的`handlers`,这里便是一个:

```go
eng.OnShutdown(func() {
    if err := daemon.shutdown(); err != nil {
        log.Errorf("daemon.shutdown(): %s", err)
    }
    if err := portallocator.ReleaseAll(); err != nil {
        log.Errorf("portallocator.ReleaseAll(): %s", err)
    }
    if err := daemon.driver.Cleanup(); err != nil {
        log.Errorf("daemon.driver.Cleanup(): %s", err.Error())
    }
    if err := daemon.containerGraph.Close(); err != nil {
        log.Errorf("daemon.containerGraph.Close(): %s", err.Error())
    }
})
```


主要进行`daemon`自身的清理工作，端口的释放，挂载点的卸载,与`graphdb`连接的关闭。

# ServeApi
```go
job := eng.Job("serveapi", flHosts...)
job.SetenvBool("Logging", true)
job.SetenvBool("EnableCors", *flEnableCors)
job.Setenv("Version", dockerversion.VERSION)
job.Setenv("SocketGroup", *flSocketGroup)

job.SetenvBool("Tls", *flTls)
job.SetenvBool("TlsVerify", *flTlsVerify)
job.Setenv("TlsCa", *flCa)
job.Setenv("TlsCert", *flCert)
job.Setenv("TlsKey", *flKey)
job.SetenvBool("BufferRequests", true)
if err := job.Run(); err != nil {
    log.Fatal(err)
}
```


查看之前在`Builtins`中注册的`handlers`表，可知`serveapi`对应的是
`apiserver.ServeApi`函数。`ServeApi`即开始监听参数中指定的各种协议和端口，并准备
开始处理`http`请求了(`docker client` 与 `daemon` 的交互都是通过`REST API`来进行
的)。




# 参考链接
1. [VFS](http://en.wikipedia.org/wiki/Virtual_file_system)
2. [How Docker container volumes work even when they aren't running?](http://stackoverflow.com/questions/24353387/how-docker-container-volumes-work-even-when-they-arent-running)
3. [Advanced Docker Volumes](http://crosbymichael.com/advanced-docker-volumes.html)
4. [Network Configuration](https://docs.docker.com/articles/networking/)
5. [Docker 源码分析（四）：Docker Daemon 之 NewDaemon 实现](http://www.infoq.com/cn/articles/docker-source-code-analysis-part4)

[1]: http://hangyan.github.io/images/posts/docker/source-3/docker-version.png "docker-version"
[2]: http://hangyan.github.io/images/posts/docker/source-3/docker-start-container.png "dsc"
[3]: http://hangyan.github.io/images/posts/docker/source-3/docker-events.png "docker-events"
[4]: http://hangyan.github.io/images/posts/docker/source-3/graph-dir.png "graph-dir"
[5]: http://hangyan.github.io/images/posts/docker/source-3/images-tree.png "images-tree"
[6]: http://hangyan.github.io/images/posts/docker/source-3/image-json.png "image-json"
[7]: http://hangyan.github.io/images/posts/docker/source-3/layersize.png "layersize"
[8]: http://hangyan.github.io/images/posts/docker/source-3/volumes.png "volumes"
[9]: http://hangyan.github.io/images/posts/docker/source-3/inspect.png "inspect"
[10]: http://hangyan.github.io/images/posts/docker/source-3/volumes-dir.png "volumes-dir"
[11]: http://hangyan.github.io/images/posts/docker/source-3/volumes-dir-data.png "data"
[12]: http://hangyan.github.io/images/posts/docker/source-3/volumes-meta.png "volumes-meta"
[13]: http://hangyan.github.io/images/posts/docker/source-3/repos.png "repos"
[14]: http://hangyan.github.io/images/posts/docker/source-3/trust.png "trust"
[15]: http://hangyan.github.io/images/posts/docker/source-3/bridge.png "bridge"
[16]: http://hangyan.github.io/images/posts/docker/source-3/docker-info.png "docker-info"
[17]: http://hangyan.github.io/images/posts/docker/source-3/native-dir.png "native-dir"
[18]: http://hangyan.github.io/images/posts/docker/source-3/state-json.png "state-json"
[19]: http://hangyan.github.io/images/posts/docker/source-3/container-json.png "container-json"
[20]: http://hangyan.github.io/images/posts/docker/source-3/daemon-log.png "daemon-log"
[21]: http://hangyan.github.io/images/posts/docker/source-3/daemon-container-handlers.png "dch"
[22]: http://hangyan.github.io/images/posts/docker/source-3/daemon-images-handlers.png "dih"
[23]: http://hangyan.github.io/images/posts/docker/source-3/containers-dir.png "containers-dir"
[24]: http://hangyan.github.io/images/posts/docker/source-3/entity.png "entity"
[25]: http://hangyan.github.io/images/posts/docker/source-3/edge.png "edge"
