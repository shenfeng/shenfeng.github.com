---
author: feng
date: '2011-10-07 20-00-00'
layout: post
status: publish
title: A web crawler, written for speed, in JAVA and Clojure
---

十一长假就快要过去了， 写的web crawler也告一段落： 速度能达到大概下载8万网页/小时， CPU和Mem的使用都比较满意：
运行40分钟的截图：

![image](/imgs/crawler-cpu-mem.png)
#### CPU， Mem使用

![image](/imgs/crawler-network.png)
#### 网络使用（4M带宽，已极限）

![image](/imgs/crawler-stat.png)
#### 按status的分布

Crawler是[Rss miner](http://rssminer.net)的一部分， git log查看， 已零星5个月， 这5个月的周末都耗在上面了， 其中大部分在crawler上， 数次大的重构或重写。

Crawer主要以[Clojure](http://clojure.org)和Java完成。 Clojure可以把程序写得很简洁， 利用Java可以很好的组织多线程， 面向对象 + functional， 感觉很不错。

开始， 我用Clojure了封装JDK 的
[URLConnection](http://download.oracle.com/javase/1.4.2/docs/api/java/net/URLConnection.html), 由于Blocking， 为了加快速度， 需要使用多线程。

#### 有一些问题， 例如：
1. 线程少速度慢， 线程多了内存受不了， 我对内存较敏感， 有一部分是想挑战自己， 也有一部分是因为我的VPS只有512M内存， 想在上面跑Rss miner, 包括一个Web server， 一个Rss fetcher, 一个Web Crawler, 一个Online的实时推荐算法， 筹划中....
2. URLConnection以[Stream](http://en.wikipedia.org/wiki/Stream_(computing)封装, 不是很方便。
3. 如果各个线程分别自己保存自己下载的数据， Disk可能比较辛苦。 如果用Queue送给单独的一个线程处理， 又有一个额外的线程开销。

我寻找 [Non-blocking](http://en.wikipedia.org/wiki/Asynchronous_I/O)的Http Client， 试用了两个， 都不太满意， 自己写了[一个](https://github.com/shenfeng/async-http-client)， 注重性能和稳定性。

#### 实现：
* 4个线程， 每个线程都是一个Loop， 相互之间是Producer， Consumer的关系， 通过Queue和Event交流
* 管理状态比较多的，用Java实现， 比如用Tagsoup抽取链接和文本， 通过规则排除部分URL
* DNS prefetch, [Pdnsd](http://en.wikipedia.org/wiki/Pdnsd)做DNS cache： UDP提前发送[Query请求](https://github.com/shenfeng/netty-http-client/blob/master/src/java/me/shenfeng/dns/DnsPrefecher.java)， 忽略结果。
<!-- * 速度可以通过信号量控制 -->
* Java搭了一个简单的框架， 提供两个Interface, 由Clojure实现

{% highlight java %}
public interface IHttpTask {
    URI getUri();
    Map<String, Object> getHeaders();
    Object doTask(HttpResponse response) throws Exception;
    Proxy getProxy();
}
{% endhighlight %}

{% highlight java %}
public interface IHttpTaskProvder {
    List<IHttpTask> getTasks();
}
{% endhighlight %}


