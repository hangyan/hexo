---

title: docker 容器的一些概念辨析
tags: [docker]
imagefeature:
comments: true
share: true
excerpt: "expose,publish/entrypoint,cmd"
---

本篇的主要内容是为了澄清 docker 容器的一些容易混淆的概念，主要分两部分，
一是容器端口的`publish`和`expose`,二是 Dockerfile 中`ENTRYPOINT`和`CMD`的区分。

<!--more-->

## Expose or Publish
`expose`的作用是为了容器间通信，也就是说 expose 的端口只能被其他容器访问，但是
不能被 docker 之外访问，而`publish`的端口即可用于容器间通信也可被外界访问。所以从
逻辑上来说,`publish`包含了`expose`。

有这样的区分主要是为了可移植性。我们知道在`Dockerfile`文件中只有`Expose`命令,
`expose`暴露的端口只能用于容器间通信，不会跟主机上的端口冲突。如果在
`Dockerfile`中加入了`publish`，在其他的机器上构建时就有可能导致端口冲突。

在命令行里使用这两个参数时，二者是可以混用的，不会冲突。同时指定`--expose`和
`--publish`与单独使用`--publish`的效果一样。

## ENTRYPOINT or CMD

Docker 提供了一个默认的`ENTRYPOINT`:`/bin/sh -c`,但是没有默认的`CMD`。比如当我们
在命令行里执行`docker run -it ubuntu bash`的时候,entrypoint 就是`/bin/sh -c`，
command 就是`bash`。

在命令行里,所有镜像名字之后的参数都是作为 command 传给了 entrypoint。在`Dockerfile`
中，我们可以指定默认的`ENTRYPOINT`和`CMD`。比如我们执行`docker run -it ubuntu`的
时候，效果和`docker run -it ubuntu bash`是一样的，因为 ubuntu 的 Dockerfile 里指定了
`CMD`为`bash`.

二者的分离主要也是应用在 Dockerfile 中,通过灵活的设置，我们可以做出来一些有趣的，
很方便使用的镜像。比如将`ENTRYPOINT`设为`[/bin/cat]`,那么在执行`docker run　catimg /etc/password`的时候就相当于在执行`/bin/cat /etc/password`,整个镜像此时变
成了一个二进制程序。这个例子可能显得有点无聊，但是既然任意程序都可以用做
`ENTRYPOINT`,自然是只有想不到，没有做不到了。如果将一个 redis 镜像的`ENTRYPOINT`设
为`["redis", "-H", "something", "-u", "toto"]`,那么在执行`docker run redisimg
get key`就相当于`docker run redisimg redis -H something -u toto get key`,这就是
一个简单明了的 redis 客户端，也不用输那么多参数了。

Dockerfile 中与此相关的命令还有一个`RUN`，相关细节较为琐碎，下面将详细叙述：

### RUN
`RUN`用来在上一层 layer 的基础上执行一些系统命令并且创建新的一层，可以说是
Dockerfile 中最为常见的命令了，主要有两种形式

- `RUN <command>` 通过`/bin/sh -c`执行
- `RUN ["executable", "param1", "param2"]` (exec 形式)

exec 形式可以避免 shell 对参数的一些处理并且可以用在一些没有`/bin/sh`的基本镜像上，
如果想使用别的 shell，也可以使用此种方式,比如`RUN ["/bin/bash", "-c", "echo hello"]`。
注意事项:

1. exec 形式是以 json 数组的形式来解析的，所以各参数必须用双引号`""`，不能用单引号`''`。
2. exec 形式下如果不明确制定是不会调用 shell 的，也就意味着一些环境变量是无法解析的，

比如`RUN [ "echo", "$HOME" ]`，如果需要必须自己明确指定所用的 shell


### CMD
`CMD`主要是为容器提供默认的运行程序，有三种形式:

- `CMD ["executable","param1","param2"]` (exec 形式)
- `CMD ["param1","param2"]` (将参数传给`ENTRYPOINT`)
- `CMD command param1 param2` (shell 形式,`/bin/sh -c`执行)

一个 Dockerfile 中只能有一个`CMD`，如果指定了多个，只有最后一个起作用。运行时的参
数会覆盖 Dockerfile 中`CMD`。

注意事项可参考`RUN`的注意事项


### ENTRYPOINT

`ENTRYPOINT`有两种形式:

- `ENTRYPOINT ["executable", "param1", "param2"]` (exec 形式)
- `ENTRYPOINT command param1 param2` (shell 形式)

类似于`CMD`,如果指定了多个`ENTRYPOINT`，只有最后一个起作用。`docker run --entrypoint`可覆盖 Dockerfile 中的设置。

exec 形式是最常用的，shell 形式会阻止任何`CMD`或者`run`的参数的执行(因为已经在
ENTRYPOINT 中指定了执行程序),但是不能传递信号。


##　参考链接
1. [Dockerfile Reference](http://docs.docker.com/reference/builder/#entrypoint)
2. [Difference between “expose” and “publish” in docker](http://stackoverflow.com/questions/22111060/difference-between-expose-and-publish-in-docker) 
3. [What is the difference between CMD and ENTRYPOINT in a Dockerfile?](http://stackoverflow.com/questions/21553353/what-is-the-difference-between-cmd-and-entrypoint-in-a-dockerfile)

