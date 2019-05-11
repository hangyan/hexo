---
categories: 技术
title: docker 源码分析(1) -- 开发环境准备
tags: [docker]
imagefeature:
comments: true
share: true
excerpt: "系列第一篇，主要为源码编译及相关环境设置"
thumb: /images/thumbs/go-helm-util1.png
---


之前看过一遍`docker`的代码，但比较粗略。第二遍准备详细过一遍并且用博客的方式将其
整理一下。目前为止`docker`的稳定版本为`1.4.1`,所以就以此版本的代码为基础进行阅读，分
析。
<!--more-->

# 本篇内容
一个大项目的源码阅读当然是从主程序入口开始最好, 一步一步，结合实际程序进行测试分
析。 本来准备一起介绍相关环境设置以及源码中关于启动部分的，但前者内容较多，所以
单列一篇进行介绍。


# 源码编译 docker

## 构建

`ubuntu`的`docker`源有 1.4.1 版本的 docker，可以直接用来与代码进行对照测试。但是能
自己手动编译一遍还是很有好处的。docker 官网上关于开发环境设置的博客
([Setting Up a Dev Environment](https://docs.docker.com/contributing/devenvironment/))
，里面已经讲的很清楚了,下面仅作简要描述，详细内容请参考官方文档。

首先看一下 docker 源码的目录结构 :

![ ][1]

下面有一个`makefile`文件和一个`Dockerfile`。直接执行`make`，会根据`Dockerfile`创
建镜像，里面包含特定版本的`golang`(由源码编译)以及其他编译 docker 所需的依赖，当前目录的 docker
源码也会被整个拷贝进镜像里用于编译。最后根据生成的镜像创建`container`用于实际的
编译并生成可执行程序。注意最后生成的可执行程序是位于`container`内部的，所以如果
要使用所编译的程序，应该使用如下命令 :

	 sudo make binary

最终生成的程序位于`./bundles/<version>-dev/binary/`,如果没有看到这些文件，一般都
是因为`BINDDIR`环境变量有问题，此时可以执行如下命令 :

	 sudo make BINDDIR=. binary

可以用生成的程序替换系统程序来方便测试 :

	 sudo service docker stop ; sudo cp $(which docker) $(which docker)_ ; sudo cp ./bundles/<version>-dev/binary/docker-<version>-dev $(which docker);sudo service docker start


## 注意事项

1. 第一次构建过程会很长，因为需要从源码构建`golang`,`llvm`以及 clone 一些依赖项目。后续再
   次构建时会复用之前生成的中间镜像，所以会很快。
   
2. 在墙内的话，一般第一次编译都会失败，因为有两个所需的网站被墙。。。第一个是
   `golang`的下载网站，这个很好解决,因为有好多网站对它做了镜像。docker v1.4.1 所
   用的 golang 版本为 1.3.3，Dockerfile 中可以看到文件名 :

		RUN     curl -sSL https://golang.org/dl/go1.3.3.src.tar.gz | tar -v -C /usr/local -xz

	在谷歌用 `Index of` 搜索此文件 :

		"Index of" go1.3.3.src.tar.gz

	找一个可用的网站替换 Dockerfile 中的 URL 即可，下面是一个可用的 :
[http://pkgs.fedoraproject.org/repo/pkgs/golang/go1.3.3.src.tar.gz/2cdbad6baefcf1007f3cf54a5bc878b7/go1.3.3.src.tar.gz](http://pkgs.fedoraproject.org/repo/pkgs/golang/go1.3.3.src.tar.gz/2cdbad6baefcf1007f3cf54a5bc878b7/go1.3.3.src.tar.gz)


	另一个被墙的便是`golang.org`,解决办法是从 github 上从 github 上下载其 fork 版本,并且用软连接或者直接拷贝到相应目录即可。具体步骤是 :
	
	- Dockerfile 构建镜像会有好多中间镜像，找到最后成功构建的那一个,用其创建一个
      `container`来进行操作

	- 从 github 上下载相关项目。我自己 fork 了一个，可以直接使用 [https://github.com/hangyan/golang.org](https://github.com/hangyan/golang.org) 

	- 做 link 或者 copy :

			mkdir -p /go/src/golang.org/ && ln -sf
            /go/src/github.com/hangyan/golang.org/x /go/src/golang.org/x

	- 提交镜像，修改 Dockerfile，将其作为新的 baseimage。中间已经成功的很多步骤也可以删掉
      了。

3. Dockerfile 中将`GOPATH` 设了两个目录,如下所示 :

		ENV     GOPATH  /go:/go/src/github.com/docker/docker/vendor
	
	原因是由于 docker 依赖项目较多，所以大部分依赖都已经在 `vendor`目录下提供，编
    译时将其设为`GOPATH`即可使用。



# 开发环境


## IDE


刚开始用`golang`的时候，好多 IDE 还在初级阶段,缺少很多必须的功能，所以就一直用
Emacs 了。IDE 基本不用配置，上手简单，嫌麻烦的可以直接使用。下面是一个常用 IDE 的列表及链接。


- LiteIDE : [sourceforge](http://sourceforge.net/projects/liteide/files/).

	感觉界面不好看。
	
- Zeus IDE : [zeus](http://www.zeusedit.com/download.html)

	windows 上的，没用过.

- Go IDE : [go-ide](http://go-ide.com/2011/08/09/goide_release_1_0_darwin.html)

	基于 Intellij IDEA 做的一个 IDE,功能比较简单。

- Golang Komodo : [Komodo](http://komodoide.com/resources/languages/komodo--golang/)

	是一个 Komodo 平台的插件。


## Emacs

有很多关于 Golang 的插件可以使用，下面是我在使用的一些插件及配置 :


### [go-mode](https://github.com/dominikh/go-mode.el)

除了基本的语法高亮及自动对齐之外，还提供了如下功能 :

- `gofmt` 集成，保存时自动排版代码

- `godoc` 集成,

- `import` 管理

- `godef` 代码跳转


	依赖`code.google.com/p/rog-go/exp/cmd/godef`项目，依旧被墙。可以从 github 上
    fork 的项目来 build,地址为[godef](https://github.com/hangyan/godef)

- `flymake` 集成，实施检查代码语法错误。


Emacs 配置示例：

	 (require 'go-mode)
	 (add-hook 'before-save-hook 'gofmt-before-save)



### [gocode](https://github.com/nsf/gocode)

提供代码自动补全功能，依赖于`autocomplete`或`company`。这个项目在 github 上的 star 数已经超过 2000,可见其流行程度。其主页上已经有关于 Emacs 的
配置，此处不再详述。

![ ][2]

### [xcscope](https://github.com/dkogan/xcscope.el)

xcscope 可以作为`godef`的一个补充项，它可以将一个目录下的代码包含的 symbols 和文件
建成一个索引数据库，然后从其中查询。原生的 xcscope 并不支持 go 语言，需要修改其索引
脚本。我现在使用的脚本链接为
[cscope-indexer](https://github.com/hangyan/Emacs/blob/master/bin/cscope-indexer)
。

![ ][5]

### [ecb](http://ecb.sourceforge.net/)

浏览代码神器，可以自己定制各窗口大小，内容 :

![ ][3]


### [helm](https://github.com/emacs-helm/helm)

Emacs 原生`M-x`增强工具，功能极多，还可以用来浏览当前文件的代码结构及跳转到相应函数或变量定
义。

![ ][4]

主要的插件就这几个，如果有兴趣，可以参考我的 Emacs 配置 :
[Emacs](https://github.com/hangyan/Emacs)。




[1]: http://hangyan.github.io/images/posts/docker/source-1/docker-root-dir.png "DockerDir"
[2]: http://hangyan.github.io/images/posts/docker/source-1/company-go.png "CompanyGo"
[3]: http://hangyan.github.io/images/posts/docker/source-1/ecb.png "ECB"
[4]: http://hangyan.github.io/images/posts/docker/source-1/helm.png "HELM"
[5]: http://hangyan.github.io/images/posts/docker/source-1/xcscope.png "Xcscope"


