---
author: feng
date: '2013-12-10 08-21-00'
layout: post
status: publish
title: gperftools 初探 - gperftools-httpd
---

相信每一个对性能感兴趣的程序员都会好奇自己的程序的瓶颈在什么地方，哪段代码是热点，哪段代码用了最多的内存。
### 初识 pprof
前段时间，用go写了一些code，受`Russ Cox`的 [Profiling Go Programs](http://blog.golang.org/profiling-go-programs)。在它的帮助下，优化程序很方便，profile线上实际运行的服务，得到真实的情况。

### gperftools初探
最近的一个项目：实时推荐，是用C++写的。靠着7年多以来，几乎每天都写代码的经验，这个程序在没有特别做优化的情况下，能在20m左右，根据用户的行为行为，算出推荐结果。20ms左右的时间，倒是没有优化的必要。
趁着项目逐渐稳定，也多出了一点闲暇时间，我也想研究一下工具。首先感兴趣的是[gperftools]( https://code.google.com/p/gperftools/) :
>Fast, multi-threaded malloc() and nifty performance analysis tools

很喜欢的介绍。试着用它跑了最近写的一个C++的线程池 [threadpool.cpp](https://github.com/shenfeng/gocode/blob/hashtable/thread_test/threadpool.cpp)，
通过submit两百万递归计算斐波那契数列第19项的jobs给线程池，来观察有多少CPU时间是花在计算上:

{% highlight sh %}
CPUPROFILE=/tmp/profile  ./threadpool
google-pprof  ./threadpool /tmp/profile
{% endhighlight %}
使用方便，只需要在编译时链接 `-lprofiler`，运行时加一个环境变量。在计算fib(19)这个函数上，占用了近95%的CPU时间。这样对自己的程序，能有个直观的认识

### gperftools-httpd初探
有了profile threadpool的甜头，很急切的想知道在线上跑的推荐程序是什么样的。在sa的帮助下，装上了相应的packages。

在gperftools 的wiki最后有这样一句话:
> Russ Cox's gperftools-httpd, a simple http server based on thttpd that enables remote profiling via google-perftool's pprof.

原来这也是Russ Cox写的。他也是go的作者之一。膜拜。

迅速checkout code，make。 它的使用也很简单，make完后，会生成libghttpd-preload.so的文件。在程序运行前，export几个环境变量就ok:

{% highlight sh %}
export GHTTPPORT=8780  #http端口号，gperftools-httpd会启一个轻量的HTTP服务器，来处理 profile请求
export LD_PRELOAD=/${DIR}/libghttpd-preload.so # DIR 为文件夹名
# start the  program
{% endhighlight %}

线上处理着真实的请求，profile线上程序(overhead可以忽略不计)：

{% highlight sh %}
google-pprof ./rcmd_server http://localhost:8780/pprof/profile
web  # 通过浏览器查看
{% endhighlight %}

这样，清楚了这个程序的瓶颈在什么地方，需要优化时，也就有了方向。
![hot spot](/imgs/gperftools_hs.png)
