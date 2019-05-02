---

title: "Scala 笔记: 函数和闭包"
modified: 2015-08-31
tags: [language]

comments: true
image:
  feature: abstract-10.jpg
excerpt: 好难的语言
redirect_from:
    - /scala/scala-notes-function-and-closures/
---

## Local functions
在函数式语言里，函数是最基本的功能块。通常为了模块清晰起见，我们需要很多 `Help
Function`，但是这些辅助函数很容易有名字冲突，暴露给外部的时候也容易引起很多问题。
Java 的主要解决方式是通过`private method`,scala 也支持这种。但是 scala 也提供了函数
式风格的解决方式: 在函数内部定义函数(`Local functions.`),就像局部变量一样,其作用
域仅限于外部函数内部。


```scala
import scala.io.Source

object LongLines {

  def processFile(filename: String, width: Int) {

    def processLine(line: String) {
      if (line.length > width)
        print(filename +": "+ line)
    }    

    val source = Source.fromFile(filename)
    for (line <- source.getLines)
      processLine(line)
  }
}
```

内部函数的一个便利之处就是它可以直接访问外部函数的变量(参数)。

## First-class functions

函数式语言与命令式语言之间最大的区别之一便是其数据和代码的一致性。在 C/JAVA 这样
的语言里,变量、类、函数、语句等时候严格区分的，但在 Lisp 这样的语言里，所有的表
达式都有值，都是数据，都可以当做变量传给函数。

scala 里有`function literal`和`function value`两个概念，有点像是`class`和
`object`之间的区别，前者都是在代码层级上而言，后者是运行时的概念。例如:

```scala
(x: Int) => x + 1
```

这是一个`fucntion literal`。你可以把它赋给一个变量并且调用它:

```go
var increase = (x: Int) => x + 1
increase(10)
increase = (x: Int) => x + 9999
increase(10)
```

简单的函数一行即可描述，多行的用`{}`包起来即可。

scala 标准库里有一个常用的`foreach`函数，它就是以一个函数为参数并且将其应用到
`collections`的各个元素之上的:

```scala
val someNumbers = List(-11, -10, -5, 0, 5, 10)
someNumbers.foreach((x: Int) => println(x))
```


`foreach`函数的参数即事一个`fcuntion literal`

## 语法糖
scala 代码看起来难懂就在于它提供了好多简化代码的方式，就拿常用于 collections 上的
`filter`函数来说(用法与 foreach 类似),因为既然是作用于`collections`,scala 有能力
自己推导出`function literal`的参数类型,比如在一个`List[String]`上操作，那么类型
自然就是`String`。简化过后可以写作:

```scala
scala> val someNumbers = List(-11, -10, -5, 0, 5, 10)
someNumbers: List[Int] = List(-11, -10, -5, 0, 5, 10)
scala> someNumbers.filter((x) => x > 0)
res5: List[Int] = List(5, 10)
```

再进一步，我们可以将参数外面的括号去掉，因为外面已经有一层括号了，里面这个就显得
有点多余了:

```scala
scala> someNumbers.filter(x => x > 0)
res6: List[Int] = List(5, 10)
```

## Placeholder

继续简化。上例中的`x`只用了一次，完全可以省略掉，只要能表示出有一个从 List 中取
出的数来参与比较即可:

```scala
scala> someNumbers.filter(_ > 0)
res7: List[Int] = List(5, 10)
```

这种占位符(`Placeholder`)语法在很多语言中都出现过，比如`python`。再看下面一个用
法:

```scala
someNumbers.foreach(println _)
```

它和下面这种形式的含义完全一样:

```scala
someNumbers.foreach(x => println(x))
```

注意在这里占位符其实代替了整个参数列表，因为它不会有引起任何混淆。

## Partially applied functions
`Partially applied functions` 也是函数式编程中常见的一个概念,在 C++中也提供了一定
的支持。以一个具有多个参数的函数来蓝本，我们可以通过提供不同数量的部分参数来创造
出不同的函数。示例如下:

```scala
scala> def sum(a: Int, b: Int, c: Int) = a + b + c
sum: (a: Int,b: Int,c: Int)Int

scala> sum(1, 2, 3)
res10: Int = 6
```

这是正常函数的调用流程。如果我们提供给 sum 两个个参数，我们就创造出了一个只需要
一个个参数的 sum 函数:


```scala
scala> val b = sum(1, _: Int, 3)
b: (Int) => Int = <function1>

scala> b(2)
res13: Int = 6
```


## Closures
闭包(Closures)的概念理解起来并不很难，但是你不把它放在函数式编程的范畴内就很难
理解它的作用。在 C 这样的语言里，闭包看起来是没什么用的。但在函数式语言里，函数经
常会表现的像一个变量，会被当做参数传来穿去。对于模块性非常好的函数——内部处理逻辑
与外部无关，只与参数有关——我们很容易理解其逻辑。但是对于需要引用到外部变量的函数，
就容易引起混淆了。这就是闭包的作用所在：它能将函数创建时的一部分环境封装起来，将
函数所需要的外部变量和其绑定在一起，以便其内部使用。看一个例子:

```scala
scala> var more = 1
more: Int = 1
scala> val addMore = (x: Int) => x + more
addMore: (Int) => Int = <function1>
scala> addMore(10)
res17: Int = 11
scala> more = 9999
more: Int = 9999
scala> addMore(10)
res18: Int = 10009
```

`addMore` 的执行需要一个外部的`more`变量，它会自动在其外部环境里找到这个变量并使
用。需要注意的是，`more`的值改变时，函数是可以感知到其变化的。

## 函数的特殊形式

### Repeated parameters
直接看例子:

```scala
scala> def echo(args: String*) =
         for (arg <- args) println(arg)
echo: (args: String*)Unit
scala> echo()

scala> echo("one")
one

scala> echo("hello", "world!")
hello
world!
```

函数的最后一个参数可以是变长的一个参数列表,使用时将其当做一个列表处理即可。需要
注意的是，函数调用时不能直接传一个列表过去，必须是将列表中的元素一个一个传进去:

```scala
scala> val arr = Array("What's", "up", "doc?")
arr: Array[java.lang.String] = Array(What's, up, doc?)

scala> echo(arr)
<console>:7: error: type mismatch;
found   : Array[java.lang.String]
required: String
      echo(arr)

scala> echo(arr: _*)
What's
up
doc?
```

### Named arguments and Default parameter values

很多语言里都有这些概念，使用命名参数可以不必按照参数声明的顺序来调用。与普通参数
混用时要把普通的参数写在前面。

默认参数和命名参数经常是结合起来使用的:

```scala
def printTime2(out: java.io.PrintStream = Console.out,
                 divisor: Int = 1) =
   out.println("time = "+ System.currentTimeMillis()/divisor)

printTime2(out = Console.err)
printTime2(divisor = 1000)
```

如果没有命名参数，想要在第一个参数为默认而第二个参数明确指定基本上是做不到的。

## 尾递归
在命令式语言中，像`while`循环这种结构，在函数式语言里一般只能通过递归来实现。看
起来更加优雅简单，但是会带来性能的损耗。尾递归便是对其的一种优化，使其能达到和
`while`循环等类似结构同样的性能.

```scala
def approximate(guess: Double): Double =
  if (isGoodEnough(guess)) guess
  else approximate(improve(guess))

def approximateLoop(initialGuess: Double): Double = {
  var guess = initialGuess
  while (!isGoodEnough(guess))
    guess = improve(guess)
  guess
}
```

注意上面的递归函数是在函数的最后一行调用自身，编译器可以将这种调用直接跳回到函数
开始处（更新参数后），省去存储各层函数堆栈的麻烦。最终编译器会对两种形式生成同样
的底层代码。

