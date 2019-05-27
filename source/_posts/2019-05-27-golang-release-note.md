---
title: Golang 1.9 Release Note
toc: true
categories: 技术
date: 2019-05-27 22:28:38
summary: 
tags:
    - Golang
    - language
---


<!-- toc -->


今天在翻存在 Pocket 里的文章，翻到一篇 Golang 1.9 的 [Release Note](https://golang.org/doc/go1.9)。不得不说，Golang 在文档化上做的相当不错。对比其他语言来说，人们关注于每个版本的更新也会更多。不过这样的对比样本比较少，可能也不是很公正。Python 因为 2/3 版本的分离，可能很多人都不太关心 3 版本的更新。C/CPP 的文档一直比较欠缺，差距太大，而且版本之间迭代的也比较慢（历史包袱也多）。

现在 Golang 的版本已经到 1.13 了大概，2 的 design 也应该有不少文档。这样追下去应该有不少发现。利用语言的新特性来优化代码也是一件很有乐趣的事情。

# 1.9 新特性介绍

## sync.Map
之前知道有这么个东西，但是自己在写代码的时候仍然没有将他作为一个`默认的选项`来考虑，仍然是自己加了 lock 手工实现，不得不说是一件很拙劣的事情。

```golang
type a struct {
  lock      sync.RWMutex
  data      map[string]interface{}
}
```

再加上一些自己实现的`get/set`函数。现在用一个普通的`sync.Map`即可。注释上说了它有自己的适用场景：

1. value 只写一次但是多次读
2. 不同 goroutines 读写的数据不重合

即使这样，适用的场景还是挺多的。更少的代码，更好的性能，值得考虑。

## Test

Golang 的 Test 包也包含了很多比较`少见`的的功能。借助 1.9 的 Release Note 看了下，也都是很实用的。

### test.Helper()

在测试中我们经常需要写一些 helper function,这些函数是服务于真正的测试函数。如果它们出错并且打印出来，只能显示到 helper function 的文件名和行号。比如我们自己实现一个 assert 函数，查看错误就很不方便。而`test.Helper()`就是为了解决这个问题，当在 helper function 中打印具体的错误时，会显示的是调用者的文件名和行号，如下面例子所示:

```go
package helper

import "testing"

func TestHelper(t *testing.T) {
	a := &asserter{T: t}
	a.Equal(1, 1)
	a.Equal(1, 2)
	a.Equal(1, 3)
}

type asserter struct {
	T *testing.T
}

func (m *asserter) Equal(a, b interface{}) {
	m.T.Helper()
	if a != b {
		m.T.Errorf("expected %#v (%T) was %#v (%T)", a, a, b, b)
	}
}
```

输出如下:

```bash
=== RUN   TestHelper
--- FAIL: TestHelper (0.00s)
        run_test.go:8: expected 1 (int) was 2 (int)
        run_test.go:9: expected 1 (int) was 3 (int)
FAIL
exit status 1
FAIL    github.com/tobstarr/gohelper    0.001s

```


### Benchmark

Benchmark 类的测试在业务上可能用处不是很大， 一般来说更适用于自定义的数据结构，比如 Golang 的标准包里的那些，需要严格测试其性能。在业务上，一般来说测试的都是具体业务逻辑，很多地方都是 mock，也没法直接测试性能，可能直接用集成测试更好。但是一旦有自定义的类似于标准包里的那些数据结构，还是挺合适的。

### Examples
有点类似于简化版的 tests，但是可以用于 godoc 的展示。也是一个一般业务上不太用到的功能。



### Subtests

应该推广的 `table driven tests`。基本上在一个 test 函数内需要多次对比结果的都应该这样写，在测试结果的输出上也更加清晰。不一定非要写成 for 循环的格式。

```go
func TestFoo(t *testing.T) {
    // <setup code>
    t.Run("A=1", func(t *testing.T) { ... })
    t.Run("A=2", func(t *testing.T) { ... })
    t.Run("B=1", func(t *testing.T) { ... })
    // <tear-down code>
}

```



# Links


1. [sync.Map](https://golang.org/pkg/sync/#Map)
2. [Better test helpers in go 1.9](https://www.tobstarr.com/2017/06/16/better-test-helpers-in-go/)
3. [testing](https://golang.org/pkg/testing/)



