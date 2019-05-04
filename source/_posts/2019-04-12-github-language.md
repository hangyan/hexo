---

title: Github 与编程语言
date: 2019-04-12 15:22:54 +0800
tags: 技术随笔

comments: true
---

一个语言的流行程度与多种因素相关,语法的简易性，package 的数量，性能，适用的场景等等。Perl 因为语法的`怪异`而逐渐无人问津，`ROR`因性能等问题也用的人越来越少。
一向处于主流地位的 C/CPP 在这几年也比不上 JavaScript 以及 Golang 的热度。在 Github 上，我们可以看到热门的项目中，JS 以及 Golang 等热门语言占据了相当瞩目的位置。
语言之间的较量从来就不是在单纯的语言层面本身，而是整个生态系统的对比。而不知不觉间，Github 也成了每个语言生态体系的一部分。

当代的主流程序员中，大多数都有 C/CPP 的语言背景。但在大部分的工作选择上，python 与 Golang 的数量越来越多，也越来越占主流。究其原因，一方面是在语法层面上，采用了与 C 相似而更简单的表达方式。但更重要的是，从一开始，他们就考虑到了包管理系统的重要性，因为它是语言生态系统的`发动机`，而 C/CPP 等语言到现在仍未有一个内建的实现。

对 Python 而言，在 Github 流行之前，pip 已经非常成熟。这让他成为了一个在任何需求面前都是被优先选择的语言之一。http 请求、web 框架、爬虫、图片解析等等，只要你想要的任何功能，都可以在 pip 里找到已经有的实现。有了 Github 之后，python 的 package 的数量的扩充进一步加快。代码在 github,二进制包在 pip,文档在 readthedocs,这是一个非常完善而让人信任的体系。

对于 Golang 来说，它就是生于 Github,流行于 Github 的。从一开始 docker/kubernetes 项目的流行，以及整个容器生态的成熟，给了 golang 一个良好的开端。go 内置的对于 github 上代码仓库的支持，即使有很多小问题，但总体来讲，这让 golang 以极快的速度建立起来了超越于 c/cpp 甚至能赶上 python/pip 的生态体系。在常规的开发场景以及业务需求上，golang 以及没有短板了。

而反观 c/cpp，在 github 之前，假设我们想要去找一个 package，只能借助于搜索引擎。也许在某些奇怪的网站上，能找到一个实现方案。但是相比于 golang/github 模式，这样的方式有明显的缺陷:

* 你不知道这个 package 的真正质量怎么样。而在 github，你可以通过 star/issues/PR 数量来推断
* 不能像 github 那样可以直接的方便地看到源码的结构,licence 等等信息
* 不能像 github 那样，一旦发现原有的 package 有需要修改的地方，直接提 PR 去调整。
* ...

这些问题导致了 c/cpp 社区倾向于不依赖于第三方实现，而是自己造轮子。或者是只能用 Boost 这样的库。整个社区就像是一个一个的孤岛，无法连接而壮大。
如何做出来一个类似 pip 或者 go get 的系统，是整个社区亟需解决的问题。

目前来看，社区已经开始了尝试，各种 package manager 已经开始小范围使用，但是离进入语言标准还有很大距离。期待能在不远的将来看到 c/cpp 借助 github 重新崛起的一天。

而与此同时，许多新语言的产生也大多都是无疾而终，如何参照现有语言的成功模式以及借助 github 的力量，也是各个语言设计者需要考虑的问题。