---
title: 代码与注释
toc: true
categories: 技术
date: 2019-07-14 21:14:49
summary:
tags: [编程]
---

代码应该拥有良好的注释一直是业界共识，毕竟，在可预见的未来里，阅读代码的主要还是人。所有的语言都支持注释，有的拥有额外的注释提取工具及格式规范。但总体来说，大部分语言在这块做的都比较一般。Python有一些约定俗称的规范以及 Sphinx 这样的工具， Golang 的规范比较简单，但内置了 godoc 工具。Lisp 算是做的比较好的，示例如下：

```commonlisp
(defun small-prime-number-p (n)
  "Return T if N, an integer, is a prime number. Otherwise, return NIL."
  (cond ((or (< n 2))
         nil)
        ((= n 2)
         t)
        ((divisorp 2 n)
         nil)
        (t
         (loop for i from 3 upto (sqrt n) by 2
               never (divisorp i n)))))
```



其独特之处在于，不同于一般语言注释位于函数体之外，而是将其放置于函数 body 开始处。这样开发者更容易将其当作函数定义的一部分，也有助于养成良好的注释习惯。



Golang 虽然 在注释方面做的普通，但是在官方 library 以及最佳实践方面成功地鼓励了人们尽可能地写非常详细的注释，具体到一个 struct 的各个字段上。尤其是在 kubernetes 社区里面,  其[Space Shuttle style](https://news.ycombinator.com/item?id=18772873) 的 Code 对于社区的蓬勃发展可以说是一个极大的背后功臣，从如下的代码便可一窥其风格：

```golang
// Adapts a ConfigMap into a projected volume.
//
// The contents of the target ConfigMap's Data field will be presented in a
// projected volume as files using the keys in the Data field as the file names,
// unless the items element is populated with specific mappings of keys to paths.
// Note that this is identical to a configmap volume source without the default
// mode.
type ConfigMapProjection struct {
	LocalObjectReference
	// If unspecified, each key-value pair in the Data field of the referenced
	// ConfigMap will be projected into the volume as a file whose name is the
	// key and content is the value. If specified, the listed keys will be
	// projected into the specified paths, and unlisted keys will not be
	// present. If a key is specified which is not present in the ConfigMap,
	// the volume setup will error unless it is marked optional. Paths must be
	// relative and may not contain the '..' path or start with '..'.
	// +optional
	Items []KeyToPath
	// Specify whether the ConfigMap or it's keys must be defined
	// +optional
	Optional *bool
}
```

可读性越好的代码越容易流行。Golang 在这方面做的很好。



Knuth 曾经发明过一种将注释发挥到极致的语言风格: `Literate Programming`。结构很类似于 Tex 和 Markdown, 代码穿插于文档之中，工具可以分别提取出其中的文档与源代码。



```bash
\section{Hello world}

Today I awoke and decided to write
some code, so I started to write Hello World in \textsf C.

<<hello.c>>=
/*
  <<license>>
*/
#include <stdio.h>

int main(int argc, char *argv[]) {
  printf("Hello World!\n");
  return 0;
}
@
\noindent \ldots then I did the same in PHP.

<<hello.php>>=
<?php
  /*
  <<license>>
  */
  echo "Hello world!\n";
?>
@
\section{License}
Later the same day some lawyer reminded me about licenses.
So, here it is:

<<license>>=
This work is placed in the public domain.
```



虽然听起来跟普通语言差别不大，但是从如上的示例上看，其观感与普通的语言的差别非常之大。文档与代码拥有了同样的重要性。整个文件更像是作者一个思路的记录与阐释。这样的好处自然是在代码的可读性性上有一些独特的优势。但缺点也同样明显，已经有无数的人从各种角度对其进行了批评。最为显著的一个问题便是，作者选定了一种顺序方式来书写代码，但是人们阅读代码的顺序与习惯则是多种多样的。有的人想要完整地阅读代码，有的只想阅读某一个片段，不同的阅读顺序，频繁地跳转，都需要语言本身的支持。而 lierateing programming 并没有在这些方面有很好的方案。也许作为简短的代码说明是一个比较适合的场景。









