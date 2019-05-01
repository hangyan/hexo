---
layout: post
title: bash各配置文件浅析
tags: [技术]
imagefeature:
comments: true
share: true
excerpt: "bash配置文件浅析"
thumb: /images/thumbs/bash.jpg
---


与bash相关的配置文件非常之多,用户目录下的`.bashrc`,`.profile`,`.bash_profile`，
系统级的`/etc/profile`等。我们也经常会发现，在某个文件里设置好了环境变量之后，并
不能总是能在使用bash时正确加载。下面将对这个问题进行深入剖析，以解除疑惑。
<!--more-->

### login shell 与 non-login shell
首先要明晰的两个概念便是`login shell`与`non-login shell` :

1. login shell

    `login shell`是指直接以`bash`命令登入系统，比如远程 ssh 或者 在文本终端登入系
    统。一般来说服务器都是这种方式，但是桌面系统不是，因为桌面系统启动时最开始的
    命令都不是 bash。但桌面系统的 `tty1-tty6`等的登入是`login shell`.
   
2. non-login shell

    `non-login shell`是指在登入系统之后打开的 shell,比如在桌面系统启动后打开的
    terminal,或者是在 bash内部再次输入 bash打开的shell.

其实二者有一个很简单的判定方法，在命令提示符下输入 `ps -f` 查看结果，示例 :

	 UID        PID  PPID  C STIME TTY          TIME CMD
	 root     37993 37901  0 21:11 pts/7    00:00:00 -bash
	 root     38109 37993  0 21:28 pts/7    00:00:00 ps -f
	 
如果显示的为`-bash`则是`login shell`,否则便是 `non-login shell`，比如在这个终端
下再次输入 `bash`之后查看 :

	 UID        PID  PPID  C STIME TTY          TIME CMD
	 root     37993 37901  0 21:11 pts/7    00:00:00 -bash
	 root     38111 37993  0 21:31 pts/7    00:00:00 bash
	 root     38119 38111  0 21:31 pts/7    00:00:00 ps -f

新的 bash 的 CMD 只是 `bash`,为 `non-login shell`。


### 配置文件加载

#### login shell
各文件加载顺序为 :

1. 如果 `/etc/profile` 存在，加载.

    此文件的内容各个发行版均不相同，且均会加载不同的其他文件。
	
2. `~/.bash_profile`
3. `~/.bashr_login`
4. `~/.profile`

    2，3，4按顺序寻找，找到第一个存在并且可读的加载。在`ubuntu`上，一般默认只有
    `.profile`存在，并且里面加载了`.bashrc`;在`slackware`上，三个都不存在;在
    `coreos`上，只有`.bash_profile`

在退出时,如果存在 `~/.bash_logout`,执行其中命令后退出


#### non-login shell
如果 `/etc/bash.bashrc` 和 `~/.bashrc` 存在，加载。


### 总结
为了统一配置文件的加载，方便用户，很多发行版都会在这些配置文件中互相调用其他的文
件，以便用户的设置在两种shell中都能得到执行。比如`ubuntu`,在 `login shell`的设置
中将 `non-login shell`的设置都加载了。但最好还是能够清楚地区分其中的区别。




