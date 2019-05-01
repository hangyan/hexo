---
layout: post
title: 现代化的开发人员实用工具
tags: [技术]
imagefeature:
comments: true
share: true
excerpt: "httpi/icdiff/slap..."
thumb: /images/thumbs/gource.jpg
---

常年混迹于linux，对命令行程序情有独钟，平时也喜欢搜集各种实用的小工具。`github`流行以来，越来越多的新的实用的开
发工具开源出来，有的可以用来替代一些老的工具，有的则是全新的。本文整理一些实用的
工具，希望大家能在实际开发中用到。
<!--more-->

# httppie
[website](http://httpie.org/) / [github](https://github.com/jakubroztocil/httpie)

一个命令行版的`http`客户端,`github`上的star数已经超过13000。可以用非常自然的语法来发
送http请求，并且彩色化输出结果。可以用来测试，调试，与`web server`交互等。官网上
有详细的文档以及示例，很容易上手.

![ ][1]


在用docker的时候用`httpie`代替`curl`与`web server`交互就非常方便,比如搜索镜像,用`curl`:

![ ][2]

用`httpie`:

![ ][3]


# icdiff
[website](http://www.jefftk.com/icdiff) / [github](https://github.com/jeffkaufman/icdiff)

意为`Improved colored diff`,可以用来替代原生的`diff`。

![ ][4]

# slap
[github](https://github.com/slap-editor/slap)

类似`sublime text`的命令行版的编辑器，感觉在服务器上使用会很方便。

![ ][5]

VS前项目经理开发的新一代IDE : [Light Table](https://github.com/LightTable/LightTable)。

![ ][21]

Github发布的 [Atom](https://github.com/atom/atom) :

![ ][22]


# jq
[website](http://stedolan.github.io/jq/) / [github](https://github.com/stedolan/jq) 

命令行版的`json`处理工具,经常用来排版输出。

![ ][6]

在没有`jq`的情况下，可以使用`python -mjson.tool`来处理，不过没有着色功能了。


# cheat
[github](https://github.com/chrisallenlane/cheat)

`man`的补充。经常想使用一个命令，但是却不知道用什么参数，`man`命令常常的英文又不
想看,这时候就可以用到`cheat`了。`cheat`不提供全面的解释，但是给出了不少常用的使
用示例，让你很快就知道用法。

![ ][7]

# gource
[website](https://code.google.com/p/gource/)

打不开的话，`github`的fork地址为: [https://github.com/acaudwell/Gource](https://github.com/acaudwell/Gource)。

Software version control visualization。可视化代码提交历史，可以用于各种分布式代
码管理系统。

![ ][8]

可以制作成非常酷炫的视频用于展示。`youtube`和优酷上应该有很多演示视频，感兴趣的
可以自行查看。我们有一次发布会就用的这个工具制作了一部分视频。



# finalterm
[website](http://finalterm.org/) / [github](https://github.com/p-e-w/finalterm)

是一个terminal工具，linux上的terminal工具多如牛毛，常见的有KDE的`konsole`,GNOME
的`gnome-terminal`，XFCE的`Terminal`,以及古老但非常强大的`xterm`，`urxvt`。这个
算起来是较新的一个,UI不错。

![ ][9]

机械领域的两大美学趋向便是蒸汽朋克与赛博朋克。如果上面的`finalterm`属于赛博朋克
流派的话，下面的两个便是蒸汽朋克风格的经典之作:

[cool-retro-term](https://github.com/Swordfish90/cool-retro-term) : 

![ ][10]

[vinterm](https://code.google.com/p/vinterm/) :

![ ][11]


# q

[website](http://harelba.github.io/q/) / [github](https://github.com/harelba/q)

在CSV或者TSV文件上执行SQL查询。

![ ][12]

[textql](https://github.com/dinedal/textql),功能类似，golang写的:

![ ][13]

# ranger
[website](http://ranger.nongnu.org/) / [github](https://github.com/hut/ranger)

比较老的一个软件了，命令行下的文件管理器,类似VI的键绑定。可扩展性很强，支持很多
种文件类型的预览。

![ ][14]


# cv
[github](https://github.com/Xfennec/cv)

显示cp,rm,dd...等命令的进展：

![ ][15]


# oh-my-zsh
[website](http://ohmyz.sh/) / [github](https://github.com/robbyrussell/oh-my-zsh)

zsh的配置文件，在github上的star数超过21000，可想而知它是多么的火。作为bash的替代
品,`zsh`提供了极强的可扩展能力。智能的自动补全，历史命令查询复用,丰富的PS1定制……
通过一些简单的配置，能够大幅度提高你的工作效率。


![ ][16]

另一个优秀的`bash`替代品是
[fish-shell](https://github.com/fish-shell/fish-shell),相对于zsh的优点是无需配置
便提供了非常丰富的功能:

![ ][17]


# babun
[github](https://github.com/babun/babun)

windows下的一个非常优秀的`terminal`工具。

![ ][18]


# impress.js
[website](http://bartaz.github.io/impress.js/#/bored) / [github](https://github.com/bartaz/impress.js)

github上star数目最多的列表中第一页就可以看到，用来做非常酷炫的PPT.Demo链接 :[Demo](http://bartaz.github.io/impress.js/#/bored)

![ ][20]


# ungit
[github](https://github.com/FredrikNoren/ungit)

使用git的便捷工具,有非常漂亮的UI，github集成。

![ ][23]

# stackedit
[website](https://stackedit.io/) / [github](https://github.com/benweet/stackedit)

浏览器里的`markdown`编辑器,功能丰富,支持与多个云存储平台的同步:

![ ][24]

# cmdlinefu
[github](http://www.commandlinefu.com/commands/browse/sort-by-votes)

最后是一个网站，有很多人分享的非常实用的命令:

![ ][19]


[1]: http://hangyan.github.io/images/posts/dev-tools/httpie.png "httpie"
[2]: http://hangyan.github.io/images/posts/dev-tools/curl-search-images.png "curl-search-images"
[3]: http://hangyan.github.io/images/posts/dev-tools/httpie-search-images.png "httpie-search-images"
[4]: http://hangyan.github.io/images/posts/dev-tools/icdiff.png "icdiff"
[5]: http://hangyan.github.io/images/posts/dev-tools/slap.png "slap"
[6]: http://hangyan.github.io/images/posts/dev-tools/jq.png "jq"
[7]: http://hangyan.github.io/images/posts/dev-tools/cheat.png "cheat"
[8]: http://hangyan.github.io/images/posts/dev-tools/gource.jpg "gource"
[9]: http://hangyan.github.io/images/posts/dev-tools/finalterm.gif "finalterm"
[10]: http://hangyan.github.io/images/posts/dev-tools/cool-retro-term.png "cool-retro-term"
[11]: http://hangyan.github.io/images/posts/dev-tools/vinterm.png "vinterm"
[12]: http://hangyan.github.io/images/posts/dev-tools/q.png "q"
[13]: http://hangyan.github.io/images/posts/dev-tools/textql.gif "textql"
[14]: http://hangyan.github.io/images/posts/dev-tools/ranger.png "ranger"
[15]: http://hangyan.github.io/images/posts/dev-tools/cv.png "cv"
[16]: http://hangyan.github.io/images/posts/dev-tools/zsh.png "zsh"
[17]: http://hangyan.github.io/images/posts/dev-tools/fish.png "fish"
[18]: http://hangyan.github.io/images/posts/dev-tools/babun.png "babun"
[19]: http://hangyan.github.io/images/posts/dev-tools/cmdlinefu.png "cmdlinefu"
[20]: http://hangyan.github.io/images/posts/dev-tools/impress.jpg "impress"
[21]: http://hangyan.github.io/images/posts/dev-tools/lighttable.png  "lighttable"
[22]: http://hangyan.github.io/images/posts/dev-tools/atom.png "atom"
[23]: http://hangyan.github.io/images/posts/dev-tools/ungit.png "ungit"
[24]: http://hangyan.github.io/images/posts/dev-tools/stack-edit.jpg "stack-edit"
