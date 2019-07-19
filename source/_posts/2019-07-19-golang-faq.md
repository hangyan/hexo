---
title: Golang FAQ
toc: true
categories: 技术
date: 2019-07-19T00:11:33.000Z
excerpt: Golang FAQ 笔记
tags:
  - Golang
  - 编程语言
---


<!-- toc -->

- [文档的可得性](#%E6%96%87%E6%A1%A3%E7%9A%84%E5%8F%AF%E5%BE%97%E6%80%A7)
- [语言设计细节](#%E8%AF%AD%E8%A8%80%E8%AE%BE%E8%AE%A1%E7%BB%86%E8%8A%82)
  * [可读性](#%E5%8F%AF%E8%AF%BB%E6%80%A7)
  * [Generic types](#generic-types)
  * [Error 与 Exception](#error-%E4%B8%8E-exception)
  * [assertions](#assertions)
  * [static link](#static-link)
- [Tips](#tips)
  * [unused imports](#unused-imports)

<!-- tocstop -->

Golang FAQ 是一篇比较好的关于 Golang 设计的文档。比较琐碎，但是解释了很多 Golang 设计的权衡之处。没有语言是完美的，从工业界来看，Golang 是一种流行的，成功的编程语言。从语言设计来看，当然仍有欠缺与不足之处。许式伟是前者的角度，王垠是后者的角度。当然除了宏观层面，FAQ 文档仍然提供了很多有趣的细节。



## 文档的可得性

所有的语言都知道支持文档。但可得性是一个更重要的维度。 Python 有 PEP 系列详细解释了语言设计的细节， 有 read the docs 系统为各种 library 提供用户文档。 Golang有 godoc 工具以及 Golang blog, 还有像 FAQ 这样的设计 detail 解释。二者在此方面都是做的比较优秀的，也在一定程度上促进了语言的流行。



## 语言设计细节

### 可读性

可读性一方面是要考虑尽量贴近大部分程序员的背景以及知识结构，另一个方面语言的设定要尽量 common, 不会让人需要额外思考来确定一个语法的语义。Golang 对于大部分具有 c / python 等背景的程序员来说非常易懂，也尽量控制新的语法的数量，所以能够迅速的大范围的流行起来。

另一方面，为了语言的可维护性，去掉一些便捷的语法糖也是一个重要的考量。Golang 去掉了 `?:` , 隐式类型转换，指针操作，Goroutine 的内部结构不对外暴露等等。虽然代码不会像 python 那样简短，但在可维护性以及稳定性上都有了改善。





### Generic types

泛型的缺乏是 Golang 广受批评的一个点。Golang 在初期为了语言的可读性以及可维护性舍弃了泛型，而且从使用角度来看，范型并不是缺少了就无法写出拥有良好结构的代码。当然现在语言发展到成熟期，泛型的加入仍是值得考虑的。



### Error 与 Exception

这两个东西只能说目前都没有完美的实现，各有各的问题。 Golang 选择了贴近于 C 语言的 error 形态，也一定程度上是为了语言结构的清晰性。



### assertions

在语言层面去掉 assert 是比较好的，因为它的使用场景主要是程序员的偷懒。但是在 test 部分仍然不提供 assert 有点说不过去。目前有很多第三方的实现。



### static link

虽然产生的 binary size 比较大，但是却正好和 Docker 成了好搭档。什么依赖都不需要 os 干涉，甚至直接从一个空的 base image 添加 go binary 也是可以执行的。



## Tips

### unused imports

尤其是在使用 Goland 的 auto-format 之类的特性时，再加上 import 时， 因为名称冲突必须使用 alias 的场景下，有时候加入一个 import 语句是一件挺麻烦的事。这种情况下，在代码修改或者重构时保留那些 unused imports 还是挺有用的，用如下的代码可以保留住这些 imports:

```go
import "unused"

// This declaration marks the import as used by referencing an
// item from the package.
var _ = unused.Item  // TODO: Delete before committing!

func main() {
    debugData := debug.Profile()
    _ = debugData // Used only during debugging.
    ....
}
```

## Links
1. [Golang FAQ](https://golang.org/doc/faq)
2. [对 Go 语言的综合评价](http://www.yinwang.org/blog-cn/2014/04/18/golang)


