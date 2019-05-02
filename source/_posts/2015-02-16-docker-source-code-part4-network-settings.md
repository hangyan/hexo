---

title: docker 源码分析(4) -- 网络设置

tags: [docker]
imagefeature:
excerpt: "Docker daemon 启动时网络设置部分"
comments: true
share: true
description: "docker network setttings"
---

本篇的主要内容是关于`docker daemon`启动时网络设置的相关部分，在上一篇中已经简要
提到([Docker daemon 启动流程](http://hangyan.github.io/docker/docker-source-code-part3-daemon-start/))
。主要内容集中在`InitDriver`函数的解析上。

<!--more-->


## 简介
`InitDriver`函数位于`github.com/docker/docker/daemon/networkdriver/bridge/driver.go`中。
主要包含参数解析,`docker0`的创建,`iptables`的设置等。

## 相关网络参数

前面两篇也已经提到了相关的网络参数，主要是以下几个:

1. iptables

    是否启用`iptables`

2. icc

    即`Inter-Container Communitaion`,是否允许`docker container`之间以及与
    `host`之间的的通信。

3. ip-masq

    是否启用`IP masquerading`,用于源地址转换。

4. bridge ip

    给`docker0`指定 IP


5. CIDR

    限制给`container`分配的 IP 范围.

6. ip

    `container`映射`port`时默认的绑定地址,默认为`0.0.0.0`.

7. bridge

    使用已经存在的网桥而不是创建`docker0`.



这些参数由命令行参数传入，最终存储在`Job`的`env`中,`InitDriver`即通过`env`来获取
相关的设置:

```go
var (
    network        *net.IPNet
    enableIPTables = job.GetenvBool("EnableIptables")
    icc            = job.GetenvBool("InterContainerCommunication")
    ipMasq         = job.GetenvBool("EnableIpMasq")
    ipForward      = job.GetenvBool("EnableIpForward")
    bridgeIP       = job.Getenv("BridgeIP")
    fixedCIDR      = job.Getenv("FixedCIDR")
)
```

之后便是对`ip`参数进行设置:

```go

defaultBindingIP  = net.ParseIP("0.0.0.0")

if defaultIP := job.Getenv("DefaultBindingIP"); defaultIP != "" {
    defaultBindingIP = net.ParseIP(defaultIP)
}
```


然后便是判断是否使用默认的网桥`docker0`:

```go
bridgeIface = job.Getenv("BridgeIface")
usingDefaultBridge := false
if bridgeIface == "" {
	usingDefaultBridge = true
	bridgeIface = DefaultNetworkBridge
}
```


## 网桥设置
确定好将要使用的网桥后,便需要对其 IP 地址进行相关设定。首先看能不能获取到其 IP 地址:
```go
addr, err := networkdriver.GetIfaceAddr(bridgeIface)
```

`docker`服务停止后`docker0`网桥并不删掉，所以如果不是第一次使用，都能成功获取
到`ip`。我们来看下首次创建`docker0`的情况:



```go
if !usingDefaultBridge {
    return job.Error(err)
}

if err := configureBridge(bridgeIP); err != nil {
    return job.Error(err)
}
    
addr, err = networkdriver.GetIfaceAddr(bridgeIface)
if err != nil {
    return job.Error(err)
}
network = addr.(*net.IPNet)
```


如果要使用自己指定的网桥而且获取不到 IP，那就只能当成错误返回了。如果是使用
`docker0`且获取不到 IP，则就创建并且赋给它 IP，`configureBridge`参数为`bridgeIp`，
即命令行里的`-bip`,一般都没有指定，默认值为空。

```go
addrs = []string{
    "172.17.42.1/16",
    "10.0.42.1/16",  
    "10.1.42.1/16",
    "10.42.42.1/16",
    "172.16.42.1/24",
    "172.16.43.1/24",
    "172.16.44.1/24",
    "10.0.42.1/24",
    "10.0.43.1/24",
    "192.168.42.1/24",
    "192.168.43.1/24",
    "192.168.44.1/24",
}

for _, addr := range addrs {
    _, dockerNetwork, err := net.ParseCIDR(addr)
    if err != nil {
        return err
    }
    if err := networkdriver.CheckNameserverOverlaps(nameservers, dockerNetwork); err == nil {
        if err := networkdriver.CheckRouteOverlaps(dockerNetwork); err == nil {
            ifaceAddr = addr
            break
        } else {
            log.Debugf("%s %s", addr, err)
        }
    }
}

```

每次给`docker0`分配 IP，都会从`addrs`一个一个地尝试，所以我们一般看到的运行中的
`docker0`的 IP 都是`172.17.42.1/16`。对`addrs`中的 IP,要检测其与`nameservers`和路由
表是否有重叠。`nameservers`即是从系统的`/etc/resolv.conf`中读取到的列表,路由表是
类似于如下图的结果中的第一列:

![ ][1]


有了 IP 之后，再次调用`GetIfaceAddr`,将获取到的地址赋给`addr`,后面的`IP Tables`的
设置要用到这个地址。


## IPTables 设置

```go
if enableIPTables {
    if err := setupIPTables(addr, icc, ipMasq); err != nil {
        return job.Error(err)
    }
}
```


下面的重点部分就是介绍`setupIPTables`函数的流程。因为所涉及到`IPTables`部分较多，
单看代码难以理顺，所以将主要结合程序运行时的日志信息来进行介绍。



### IP Masquerade

关于`IP Masquerade`的介绍,因为个人英语水平有限，怕翻译的不准确，感兴趣可以先看下这个
简单的介绍[What is IP Masquerade?](http://www.tldp.org/HOWTO/IP-Masquerade-HOWTO/ipmasq-background2.1.html)。
简单来说就是用来做对包的源地址做地址转换(NAT)

如果`ipmasq`参数为真，则插入一条相应的 rule(先检查是否存在，不存在则插入)
```go

natArgs := []string{"POSTROUTING", "-t", "nat", "-s", addr.String(), "!", "-o", bridgeIface, "-j", "MASQUERADE"}

if !iptables.Exists(natArgs...) {
    if output, err := iptables.Raw(append([]string{"-I"}, natArgs...)...); err != nil {
        return fmt.Errorf("Unable to enable network bridge NAT: %s", err)
    } else if len(output) != 0 {
        return &iptables.ChainError{Chain: "POSTROUTING", Output: output}
    }
}

```

从日志中我们可以看到实际运行的命令是：
![ ][2]

这条 rules 的作用是对于 container 向外发出的包做源地址转换操作.

### Package Forward
- Inter-Container Communitaion

如果启用 Inter-Container Communitaion (`icc`为真),则实际执行的结果如下:
![ ][3]

确保`container`之间包的转发的`target`为`ACCEPT`。

```go
iptables.Raw(append([]string{"-D"}, dropArgs...)...)
if !iptables.Exists(acceptArgs...) {
    log.Debugf("Enable inter-container communication")
    if output, err := iptables.Raw(append([]string{"-I"}, acceptArgs...)...); err != nil {
        return fmt.Errorf("Unable to allow intercontainer communication: %s", err)
    } else if len(output) != 0 {
        return fmt.Errorf("Error enabling intercontainer communication: %s", output)
    }
}
```

- Non-Intercontainer Outgoing Packets

对于`non-intercontainer outgoing packets`,也将其`target`设为`ACCEPT`:
```go
outgoingArgs := []string{"FORWARD", "-i", bridgeIface, "!", "-o", bridgeIface, "-j", "ACCEPT"}
if !iptables.Exists(outgoingArgs...) {
	if output, err := iptables.Raw(append([]string{"-I"}, outgoingArgs...)...); err != nil {
		return fmt.Errorf("Unable to allow outgoing packets: %s", err)
	} else if len(output) != 0 {
		return &iptables.ChainError{Chain: "FORWARD outgoing", Output: output}
	}
}
```

实际执行的命令为:
`/sbin/iptables --wait -C FORWARD -i docker0 ! -o docker0 -j ACCEPT`

`iptables`的参数中,`-C`代表检查,`-D`删除,`-I`插入。一般设置都是先检查，没有的话
再插入，所以一般日志中见到的都只有`-C`

- Incoming Packets for Existing Connecting

Accept incoming packets for existing connections

```go
existingArgs := []string{"FORWARD", "-o", bridgeIface, "-m", "conntrack", "--ctstate", "RELATED,ESTABLISHED", "-j", "ACCEPT"}

if !iptables.Exists(existingArgs...) {
    if output, err := iptables.Raw(append([]string{"-I"}, existingArgs...)...); err != nil {
        return fmt.Errorf("Unable to allow incoming packets: %s", err)
    } else if len(output) != 0 {
        return &iptables.ChainError{Chain: "FORWARD incoming", Output: output}
    }
}
```

实际执行的命令为:
`/sbin/iptables --wait -C FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEP`

### Docker Chain
以上两部分就是`setIPTables`的流程,主要是一些基本的网络设置。除此之外,还有一些针
对`NAT`表中`docker chain`的设置。

先从日志中看看实际执行的命令是什么:

![ ][4]


(`-F` 删除一条`chain`中的所有规则,`-X`删除一条用户自定义的`chain`,`-N`创建一条用
户自定义的`chain`,`-A`在一条`chain`后面添加一条规则.)

首先是尝试删除`docker chain`中的规则及其本身:
```go
if err := iptables.RemoveExistingChain("DOCKER"); err != nil {
    return job.Error(err)
}
```

然后添加新的空的`docker chain`：

```go
if enableIPTables {
    chain, err := iptables.NewChain("DOCKER", bridgeIface)
    if err != nil {
        return job.Error(err)
    }
    portmapper.SetIptablesChain(chain)
}
```


最终的`nat`的表的结果如下图所示:
![ ][5]

当有`container`的端口暴露时，我们就可以看到其中会有新的`rules`添加进来:

![ ][6]



## Handlers
网络设置的最后一部分是一些`handler`的注册，具体见下面列表:

| Name | Funciton | Description |
|:----:|:--------:|:-----------:|
|allocate_interface | Allocate | Allocate a network interface | 
| release_interface | Release |  Release an interface for a select ip |
| allocate_port | AllocatePort | Allocate an external port and map it to the interface |
| link | LinkContainers | |


[1]: http://hangyan.github.io/images/posts/docker/source-4/ip-route.png "ip-route"
[2]: http://hangyan.github.io/images/posts/docker/source-4/ip-masq.png "ip-masq"
[3]: http://hangyan.github.io/images/posts/docker/source-4/icc.png "icc"
[4]: http://hangyan.github.io/images/posts/docker/source-4/docker-chain.png "docker-chain"
[5]: http://hangyan.github.io/images/posts/docker/source-4/nat-table.png "nat-table"
[6]: http://hangyan.github.io/images/posts/docker/source-4/docker-chain-new-rules.png "docker-chain-new-rules"
