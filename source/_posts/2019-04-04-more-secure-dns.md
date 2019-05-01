---
layout: post
title: 更安全的 DNS
date: 2019-04-04 20:56:03 +0800
tags: 技术
excerpt: 受信任的 DNS 解析服务器 / DoH /  QNAME minimization
comments: true
---


<!-- toc -->

HTTPS 已经越来越普及了，包括 Github Pages 都可以很方便的集成。浏览器也开始通过各种手段来促进网站都尽量使用 HTTPS。相比来说，
DNS 的安全性受重视的程度就没那么高。Firefox 写了一个一篇非常好的介绍文章(见最后Links)，系统性地介绍了当前的问题以及解决方案。在这里整理一下。

## 主要问题

DNS协议主要交流的信息就两个： ip 和 domain, 这两个东西一般来说，很不起眼，但是漏洞(非安全协议)摆在这，总会有各种不法分子来作恶。目前我能想到的比较明显的两个就是

* 运营商劫持： 一个不靠谱的运营商，在各个网站插入各种恶心的广告, 重定向网址等
* 不安全的公共网络: 比如公共场所的 wifi, 可以伪造各种网站，收集用户的上网记录等等

火狐也在文章中写出了 dns 查询过程中可能出现的安全漏洞：

![](https://hacks.mozilla.org/files/2018/05/03_04-768x383.png)

主要有三个

1. 不可靠的本地 dns 服务器
2. 中间人攻击
3. 一些根域名服务器能追踪你的解析记录。


## 解决方法

这几个问题，火狐都列出了解决方法。

### 使用受信任的DNS解析服务器

一般用户，包括互联网从业者，都不太关心本地配置的 dns 服务器是什么,一般都是运营商默认的。一般人也没有精力(能力)在出问题的时候去根运营商协调解决。火狐的思路是在用户使用浏览器的时候（不干涉其他程序），选择火狐信任的 dns 服务器来进行 dns 查询（可配置为失败时fallback到机器默认的）。目前火狐是和 cloudflare 合作，由 cloudflare 来提供域名解析服务，未来应该会有更多的提供商。

目前的 Firefox 66 版本已经支持了此选项（不一定是最早支持版本）,可以在 `about:config` 里面搜索 `trr` 查询到:


![](https://hangyan.github.io/images/posts/dns/trr.png)

`mode` 设为 2 即是 fallback 模式，火狐会首先尝试用自己信任的 dns server 来解析域名，失败时用用户系统的。

这是一个比较好的思路，首先对大部分用户来说，常见的上网操作都是浏览器提供的，能解决大部分人大多数的问题。第二，cloudflare 这样的厂商属于比较有责任感的，愿意提供一些非常有益于公众的技术服务。未来火狐能接入更多的厂商，那么在可靠性和中立性上都有了很大的保障


### DoH

Doh 的全称是 `DNS over HTTPS`, 是用来解决上面所属的第二个问题。HTTPS 已经在其他方面获得了很广泛的使用，用在这里也是非常自然的一个解决方案。火狐在选择受信任的 dns 解析服务商时，`DoH`也是一个基本要求。互联网的很多网络技术都存在了很多年，有提升的空间，但要考虑到对现有网络的兼容性，所以技术的改进是非常缓慢的。如果要有所有的 ISP 都支持 DoH 也是不现实的。但现在火狐的解决方法避免了这个问题，因为只要火狐在向 cloudflare 发送查询的时候使用 https 就行了，用户只需信任浏览器就好。这是一个非常难得的对现有互联网环境的改善。


### QNAME minimization

第三个问题的主要原因就是，在现有的 dns 查询方式上，每次查询，我们都需要将完整的域名发送出去。我们知道 dns 的查询方式基本上是一个递归查询, 比如 `en.wikipedia.org`, 这个域名, 其查询的流程如下：

* `Root DNS`: Root DNS 会告知在哪可以查询到 `.org` 相关的信息
* `Top Level Domain`: 也叫 `TLD`。这个 TLD 知道所有的以 `org` 结尾的域名。比如 `wikipedia.org`,但再往下的子域名就不知道了
* `Authoritative Server`: wikipedia的域名服务器。他知道 `wikipedia.org` 下面的子域名的详细信息

目前的实现中，每一次的查询都需要把整个 `en.wikipedia.org` 发送到相应的服务器上。但从逻辑上来讲，其实每次只发相应的部分就可以了。比如发给 `Root DNS`的信息是 `org`, 发给 `TLD` 的是 `wikipedia`,发给 `Authoritative Server` 的是 `en` 即可。这样做的好处有；

1. 减少数据传输量
2. 中间人和 DNS Server 能够 track 的信息都减少了

目前的 DNS 协议规范里也并没有强制要求每次的查询必须发全量的名字。所以目前来看是一个非常有吸引力的优化方案。

一个已知的问题是 `zone cut`, 虽然没找到具体的定义。但是通过下面的例子可以很容易理解这个问题。因为客户端(本地的 resolver)现在承担了分解域名的责任，那么下面一个域名

```yaml
www.foo.bar.exmaple
```
在发送给 `example` 查询的数据的时候，有两种可能

* `bar.example`
* `foo.bar.example`

因为 resolver 不知道具体的 zone cut 信息，所以它很可能会需要多查询一次。不过本地 resolver 一般都会维护 cache 信息，所以性能上还是可以接受的。





## 其他问题

还有一个问题就是，即使我们在上面用了种种措施来进行了安全的 dns 解析，查询到了 ip ,在请求真实的 server 时，我们仍然需要发送完全的域名给 server 。因为服务器有可能同时 host 了多个 domain（virtual hosting）。这个请求是不加密的，因为 TLS 建立连接的时候需要这个名字（先有鸡还是先有蛋的问题，具体的解释请查询`SNI`）。

这个薄弱的一环暂时还没有什么特别好的方法来解决，但是可以通过其他的技术来减少这样的请求

* `HTTP2`: http2 可以链接复用。同一个 host 上，只要与其中一个 server 建立了链接，其他的就不需要再发送这个请求了
* `CDN`: 越来越多的网站开始使用CDN(github pages的模式也很类似于这个)。只要与其中一个站点建立链接，其他的都不需要了



## Links

* [A cartoon intro to DNS over HTTPS](https://hacks.mozilla.org/2018/05/a-cartoon-intro-to-dns-over-https/)
* [How to Enable DNS-over-HTTPS (DoH) in Mozilla Firefox](https://www.trishtech.com/2018/08/how-to-enable-dns-over-https-doh-in-mozilla-firefox/)
* [DNS Query Name Minimisation to Improve Privacy](https://datatracker.ietf.org/doc/rfc7816/?include_text=1)
* [What is SNI](https://www.thesslstore.com/blog/what-is-sni/)



