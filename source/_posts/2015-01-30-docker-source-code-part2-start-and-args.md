---
layout: post
title: docker 源码分析(2) -- 主程序及命令行参数解析
tags: [docker]
imagefeature:
comments: true
share: true
excerpt: 从主函数入口解析启动流程及参数
---


研究一个大项目的源码，最好是从`main`函数入口，一步一步与实际程序相结合，将实际代
码与相应相印证。所以本篇的主要内容就是 docker 主程序的启动流程以及命令行参数解析的
过程。
<!--more-->

# 程序入口

docker 的`main`函数位于`github.com/docker/docker/docker/docker.go`文件中。代码不
是很长，逻辑也比较简单，主要的内容便是解析命令行参数并且进行各项设定，启动
`daemon`等。下面将详述各个部分。


# 命令行参数解析
支持命令行参数几乎是所有 cmd 程序所必备的功能，先不看 docker 的代码，依据我们过去的
经验，应该不难理出命令行参数解析的一般流程 :

1. 设定好程序所支持的命令行参数列表，长选项、短选项、数据类型、默认值、描述信
   息……等

2. 一个一个解析实际输入的参数，获取实际值。其中要考虑短选项的组合、错误的参数、
   出错的提示信息……

在 docker 的`main`函数中，命令行参数解析的功能主要由`mflag`包提供，而在`main`里只
需要这一句调用 :

```go
import (
	flag "github.com/docker/docker/pkg/mflag"
	...
)

flag.Parse()
```


看函数名的意思，应该就是直接开始解析了。那解析前的设定在哪呢？

## init 设定

在 golang 中,`main`并不总是最早开始执行的代码。在执行一个`package`中的代码的时候，需
要先初始化其`package-level`的变量以及执行`init`函数，如果有的话。如果导入了其他
的包，也要先对其进行初始化。在`docker`中,命令行参数的初始化设定即是通过包内的变量及
`init`函数来进行的。


`github.com/docker/docker/docker/flags.go`:

``` go
flVersion     = flag.Bool([]string{"v", "-version"}, false, "Print version information and quit")
flDaemon      = flag.Bool([]string{"d", "-daemon"}, false, "Enable daemon mode")
flDebug       = flag.Bool([]string{"D", "-debug"}, false, "Enable debug mode")
...
flLogLevel    = flag.String([]string{"l", "-log-level"}, "info", "Set the logging level")
```

在命令行里输入`docker`看下:

![arg-1](http://hangyan.github.io/images/posts/docker/source-2/arg-1.png)

可以看到结果和代码是一一对应的。

各个参数的设定都是类似的，长/短选项，默认值,描述信息。进入其中一个函数看看：

`github.com/docker/docker/pkg/mflag/flag.go`:

```go
 func Bool(names []string, value bool, usage string) *bool {
	 return CommandLine.Bool(names, value, usage)
 }
```

各个参数依其数据类型分类,我们先看看`CommandLine`是什么 :

`github.com/docker/docker/pkg/mflag/flag.go` :

```go
 var CommandLine = NewFlagSet(os.Args[0], ExitOnError)

 func NewFlagSet(name string, errorHandling ErrorHandling) *FlagSet {
	 f := &FlagSet{
		 name:          name,
		 errorHandling: errorHandling,
	 }
	 return f
 }
```

`CommandLine`属于`package-level`的变量，在被 docker 的`main`包导入时就已经初始化好
了。`CommandLine`就是我们预设命令行参数的存储地方，其类型为`FlagSet`。

## FlagSet

`Flagset`存储了我们预先设定好的所支持的命令行参数信息，并且在解析过程中动态更
新 : 

`github.com/docker/docker/pkg/mflag/flag.go` :

```go
 type FlagSet struct {
	 Usage func()
	 name          string
	 parsed        bool
	 actual        map[string]*Flag
	 formal        map[string]*Flag
	 args          []string
	 errorHandling ErrorHandling
	 output        io.Writer 
}
```

1. `Usage` : 见名知意，可知是用来输出`help`信息的。一般是在没输入参数或者参数出错
   的时候使用

2. `name` : `CommandLine`将其设为`docker`(`os.Args[0]`).
3. `parsed` : 是否已经解析完成
4. `actual` : 实际解析出来的命令行参数结果.
5. `formal` : 存储预先定义好的所支持的命令行参数信息。
6. `args`   : 输入的命令行参数列表
7. `errorHandling` : 解析遇到错误时的处理方式。

每一个命令行参数所对应的结构体为`Flag`,其定义为 :

`github.com/docker/docker/pkg/mflag/flag.go` :

```go
 type Flag struct {
	 Names    []string // name as it appears on command line
	 Usage    string   // help message
	 Value    Value    // value as set
	 DefValue string   // default value (as text); for usage message
 }
```

定义简单明了，`Names`为`string`列表是因为很多参数既有长类型也有短类型，两个名字
都存下来。还有一些个别的情况时将要废弃的参数形式，比如:

```go
 flEnableCors  = flag.Bool([]string{"#api-enable-cors", "-api-enable-cors"}, false, "Enable CORS headers in the remote API")
```
其名字前有一个`#`符号。在解析时如果遇到这类参数，会输出一些警告信息 :

`github.com/docker/docker/pkg/mflag/flag.go#parseOne()` :

```go
for i, n := range flag.Names {
	if n == fmt.Sprintf("#%s", name) {
		replacement := ""
		for j := i; j < len(flag.Names); j++ {
			if flag.Names[j][0] != '#' {
				replacement = flag.Names[j]
				break
			}
		}
		if replacement != "" { // 内容过长，省略部分。
			fmt.Fprintf(f.out(), "Warning: '-%s' is deprecated, ...)
		} else {
			fmt.Fprintf(f.out(), "Warning: '-%s' is deprecated, ...)
		}
    }
}  

```
效果如下图所示 :

![deprecated](http://hangyan.github.io/images/posts/docker/source-2/deprecated.png)





## 设置过程

这里说的设置过程是指在`package`初始化时所设定的支持的命令行参数的过程，以前面提
到过的`daemon`参数为例。

```go
 flDaemon      = flag.Bool([]string{"d", "-daemon"}, false, "Enable daemon mode")
```

前面已经提到过，`flag`大部分的操作都由`CommandLine`变量执行,调用结果为:
```go
func (f *FlagSet) Bool(names []string, value bool, usage string) *bool {
	p := new(bool)
	f.BoolVar(p, names, value, usage)
	return p
}
```

基本上还是向下传递参数，只不过多了一个`p`,`p`是一个指针，`flDaemon`和`p`指向同一
个地址，即参数的值。前面说过在解析过程中会动态更新参数的值，用指针既可保证
`flDaemon`指向的是最后实际解析出来的值。

```go
func BoolVar(p *bool, names []string, value bool, usage string) {
	CommandLine.Var(newBoolValue(value, p), names, usage)
}

type boolValue bool
	
func newBoolValue(val bool, p *bool) *boolValue {
	*p = val
	return (*boolValue)(p)
}
```

type 定义了一个新的类型`boolValue`,可以猜想处理`string`会有`stringValue`,处理`in`会有`intValue`...它们都实现了`Value`接口:

```go
type Value interface {
	String() string
	Set(string) error
}
```

`Value`是用于动态存储位于`Flag`中的值的接口.想想看,在最开始解析命令行参数时,我们需要对不同的类型作分别处理,有了统一的`Value`接口,
后续的处理就可以统一进行,不用对每种类型都定义处理函数.

以`int`为例,其`intValue`各接口定义如下:

```go
type intValue int

func newIntValue(val int, p *int) *intValue {
	*p = val
	return (*intValue)(p)
}

func (i *intValue) Set(s string) error {
	v, err := strconv.ParseInt(s, 0, 64)
	*i = intValue(v)
	return err
}

func (i *intValue) Get() interface{} { return int(*i) }

func (i *intValue) String() string { return fmt.Sprintf("%v", *i) }

```

其他类型与此类似，稍有不同的是`bool`类型。因为`bool`类型的参数通常并不需要明确地
指明其值，只要参数出现，即可认为为`true`。比如`-d`参数,并不需要写`-d=true`。针对
这种情况,`boolValue`提供了额外的`IsBoolFlag()`函数和`boolFlag` interface.

```go
func (b *boolValue) IsBoolFlag() bool { return true }

type boolFlag interface {
	Value
	IsBoolFlag() bool
}
```


再回到原来的处理流程，看看最终的`Var`函数的实现 :

```go
func (f *FlagSet) Var(value Value, names []string, usage string) {
	flag := &Flag{names, usage, value, value.String()}
	for _, name := range names {
		name = strings.TrimPrefix(name, "#")
		_, alreadythere := f.formal[name]
		if alreadythere {
			var msg string
			if f.name == "" {
				msg = fmt.Sprintf("flag redefined: %s", name)
			} else {
				msg = fmt.Sprintf("%s flag redefined: %s", f.name, name)
			}
			fmt.Fprintln(f.out(), msg)
			panic(msg) 
		}
		if f.formal == nil {
			f.formal = make(map[string]*Flag)
		}
		f.formal[name] = flag
	}
}
```

整个逻辑比较简单，先生成相应的`Flag`变量，然后建立各个参数名(长短名，将要废弃的
名字)对其的映射。各个参数均以此流程设置，最后都存储在`FlagSet`的`formal`映射表中，
后续的解析便可以对照着处理了。


## 解析过程

```go
func Parse() {
	CommandLine.Parse(os.Args[1:])
}
```


解析过程和设定过程一样都是由`CommandLine`变量来执行的，`Parse`直接读取全部参数
(除`Args[0]`即`docker`外)进行处理 :


![parse](http://hangyan.github.io/images/posts/docker/source-2/parse.png)

首先将`parsed`置为`true`,然后将所有参数存入`CommandLine`的`args`,之后便是逐个处
理参数，在`for`循环内一直调用`parseOne`，处理出错的参数，依据`errorHandling`的设
置来决定是继续还是退出等等。我们先看看`parseOne`的实现,因为函数代码过长，分段详述:

1. 先判断是不是一个 flag

```go
if len(f.args) == 0 {
	return false, "", nil
}
s := f.args[0]
if len(s) == 0 || s[0] != '-' || len(s) == 1 {
	return false, "", nil
}
if s[1] == '-' && len(s) == 2 { // "--" terminates the flags
	f.args = f.args[1:]
	return false, "", nil
}
name := s[1:]
if len(name) == 0 || name[0] == '=' {
	return false, "", f.failf("bad flag syntax: %s", s)
}

```


`len(f.args) == 0` 一般代表解析的终止,没有更多的参数了，结合上述`Parse`函数中的
判断，此时就会跳出`for`循环，正常结束解析流程。其他的几种`args[0]`情况也会导致相同结果:

- 长度为 0 或 1
- 不以`-`开头
- 值为 `--`
- 格式错误，比如`-=`之类的。

如果确定`args[0]`是一个 flag,其会从`f.args`去除，以便下一次处理的`args[0]`是下一
个参数。如果`args[0]`是形如`--debug=false`的格式，便需从中取出相应的`name`和
`value`：

```go
f.args = f.args[1:]
has_value := false
value := ""
if i := strings.Index(name, "="); i != -1 {
	value = trimQuotes(name[i+1:])
	has_value = true
	name = name[:i]
}
```

有了`name`和`value`后，便可以与之前存在`f.formal`中的参数列表相对照,看其是否属于
程序所支持的参数:

```go
flag, alreadythere := m[name] // BUG
if !alreadythere {
	if name == "-help" || name == "help" || name == "h" { 
		f.usage()
		return false, "", ErrHelp
	}
	if len(name) > 0 && name[0] == '-' {
		return false, "", f.failf("flag provided but not defined: -%s", name)
	}
	return false, name, ErrRetry
}


```


前面提到过`CommandLine`的`errorHandling`是`ErrorOnExit`，碰到错误会直接退出。如
果没有在`formal`表中找到相应的记录,有三种情况，一种是需要查看帮助信息，
系统就在打印好帮助信息后退出，另一种是程序不支持的参数，打印错误信息退出。最后一
种是短参数写在了一起，比如`-dD`,代表`--daemon --debug`,这种情况需要返回上层继续
处理，我们也可以看到在`Parse`函数中对`ErrRetry`作了单独处理,将参数字符串分割为单
个字母，然后分别解析。


之后需要对`bool`类型的参数做特殊处理，原因前已详述:

```go
if fv, ok := flag.Value.(boolFlag); ok && fv.IsBoolFlag() { 
	if has_value {
		if err := fv.Set(value); err != nil {
			return false, "", f.failf("invalid boolean value %q for  -%s: %v", value, name, err)
		}
	} else {
		fv.Set("true") //默认为 true
	}
}
```

对于其他的类型，则必须解析到其值:

```go
else {
	// It must have a value, which might be the next argument.
	if !has_value && len(f.args) > 0 {
		// value is the next arg
		has_value = true
		value, f.args = f.args[0], f.args[1:]
	}
	if !has_value {
		return false, "", f.failf("flag needs an argument: -%s", name)
	}
	if err := flag.Value.Set(value); err != nil {
		return false, "", f.failf("invalid value %q for flag -%s: %v", value, name, err)
	}
}
```

一般情况下都是以`args`列表中的下一个字符串为其值。至此，一个参数解析流程将结束，
后面只要不断重复此过程即可，除了需要对将要废弃的参数打印一些警告信息。当处理结束
时,`CommandLine`的`formal`映射表中包含了所有预设的参数及更新的值，`actual`表中只
包含了程序运行时实际使用的参数及其信息。



# 后续设定

除了参数解析，整个`main`函数的其他部分就是比较简单地用解析到的值设置各个组件，理解了前者之后，后面的部分就没有什么难点了。

## 版本号
```go
if *flVersion {
	showVersion()
	return
}
```

代码其实没什么好说的，这里主要想提及的是`docker`里设置版本号的方式。在`docker`根
目录下会有一个`VERSION`文件，里面记录了程序的版本号，然后在编译脚本中会读取其内
容来进行设置:

`github.com/docker/docker/hack/make.sh` :
```bash
VERSION=$(cat ./VERSION)
```

## 日志级别

```go
if *flLogLevel != "" {
	lvl, err := log.ParseLevel(*flLogLevel)
	if err != nil {
		log.Fatalf("Unable to parse logging level: %s", *flLogLevel)
	}
	initLogging(lvl)
} else {
	initLogging(log.InfoLevel)
}

if *flDebug {
	os.Setenv("DEBUG", "1")
	initLogging(log.DebugLevel)
}
```

有两个设置日志的参数 : `--log-level` 和 `--debug`，后者只是为了方便使用，且优先级更高。
	

## sockets
```go
if len(flHosts) == 0 {
	defaultHost := os.Getenv("DOCKER_HOST")
	if defaultHost == "" || *flDaemon {
		// If we do not have a host, default to unix socket
		defaultHost = fmt.Sprintf("unix://%s", api.DEFAULTUNIXSOCKET)
	}
	defaultHost, err := api.ValidateHost(defaultHost)
	if err != nil {
		log.Fatal(err)
	}
	flHosts = append(flHosts, defaultHost)
}
```

`docker daemon`可以监听三种类型的`socket` :

1. unix

   由上面代码可知，这是默认的形式 ，位于 `/var/run/docker.sock`。
   
2. tcp

   远程访问(`web api`)需要开启`tcp socket`,默认是不加密和无需认证的。如果需要监
   听所有`interface`,可以设为`-H tcp://0.0.0.0:2375`,或者可以自己指定特定的 IP。

3. fd

   基于`systemd`的系统可以用到，便于其他服务通过`systemd socket activation`与
   `docker daemon`交互。详见:[sockert activation](http://0pointer.de/blog/projects/socket-activation.html)。

`flHosts`是一个列表，可以多次指定`-H`参数。

## daemon

```go
if *flDaemon {
	mainDaemon()
	return
}
```

`docker`并不像`redis`等程序那样分为`server`和`client`程序，区别即在这里。如果有`-d`参
数，就以`daemon`方式启动，没有，就当做是`client.`,然后继续解析子命令及其参数进行
处理。后面介绍的流程就是只针对`client`而言。

## TLS 认证

即使对 TLS 的原理不是很了解，通过下面的代码，也很容易理解`docker`的认证过程:


![tls](http://hangyan.github.io/images/posts/docker/source-2/tls.png)

与此相关的主要有四个参数 :


![tls-args](http://hangyan.github.io/images/posts/docker/source-2/tls-args.png)

如果`--tlsverify`或者`--tls`为`true`,则启用 TLS 认证。通过指定的三个 PEM 文件，生成
一个`tls.Config`,用于后面的`docker client`与`docker daemon`的连接。

## DockerCli

```go

protoAddrParts := strings.SplitN(flHosts[0], "://", 2)

if *flTls || *flTlsVerify {
	cli = client.NewDockerCli(os.Stdin, os.Stdout, os.Stderr, nil, protoAddrParts[0], protoAddrParts[1], &tlsConfig)
} else {
	cli = client.NewDockerCli(os.Stdin, os.Stdout, os.Stderr, nil, protoAddrParts[0], protoAddrParts[1], nil)
}

```

## 子命令
最后一部分便是`docker client`子命令的解析与执行，比如`doker ps`,`docker stop <id>`等等。具体细节就留待以后解析了。

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






