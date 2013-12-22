---
author: feng
date: '2013-12-22 08-21-00'
layout: post
status: publish
title: gperftools-httpd分析server性能杂记
---

## 前记

[Redis](http://redis.io/) 相信很多人很熟悉了：广泛使用key-value
store，提供方便的访问Hash，List，Set等接口。Redis的通讯协议简洁，高效，并且客户端成熟。

最近在写一个程序[shenfeng/listdb](https://github.com/shenfeng/listdb)，C++实现的一个Redis协议的server，封装facebook
的[rocksdb](http://rocksdb.org/)，提供基于磁盘的List，Hash API(比如RPUSH，LRANGE，HSET，HGET，[参见Redis的文档](http://redis.io/commands#list))。

rocksdb继承于Gogole 的leveldb，是一个磁盘key-value lib，并提供了[Merge-Operator](https://github.com/facebook/rocksdb/wiki/Merge-Operator)
的支持，为实现高效率的基于磁盘List， Hash API创造了条件。

`listdb`将是推荐系统的基础服务，提供以下功能：

1. 存储每个user的历史行为：user和item的关系
2. 快速提取user的行为：用户的历史决定了这个用户，是个性化算法的基础
3. 定期dump <用户，行为>矩阵，用于离线模型训练

*曾用redis来存储用户的历史行为，无奈存储了10亿左右<用户，行为>对后，内存消耗大（20G），向公司要机器困难（相比于写code），嗨，公司对个性化推荐这块不舍得资源投入。无奈之余，花周末时间，写个listdb从根上解决内存资源少的问题。*

由于Redis的协议比较简单，今天周末，在家用C++实现了协议的Decode和Encode（有限状态机的方式，[redis_proto.hpp](https://github.com/shenfeng/listdb/blob/master/src/redis_proto.hpp)），网络库采用的Redis的ae.c，server已经能跑起来了。用reids提供的redis-benchmark工具测试了一下性能，和redis-server相差不大，在i7-2600上，每秒能搞定200k次get，set请求。

一时兴起，按照前一篇blog[gperftools 初探 -
gperftools-httpd](http://shenfeng.me/gperftools-gperftools-httpd.html)介绍的方法，用gperftools跑了一下我的code和redis
code:

### profile脚本

{% highlight sh %}
# 启动redis-server，会启动一个http server，监听在9999端口，使得可以用pprof分析性能
export LD_PRELOAD=/home/feng/workspace/gperftools-httpd-read-only/libghttpd-preload.so
./src/redis-server

# 发出1千万get请求，50个长连接
./src/redis-benchmark -p 7389 -t get -n 10000000

# 分析性能，生成svg调用关系和cost图
google-pprof ./src/redis-server 'http://localhost:9999/pprof/profile'
{% endhighlight %}

### profile 结果  

1. [点击查看`redis-server`的profile结果](/imgs/redis-server-profile.svg)
2. [点击查看`listdb-server`的profile结果](/imgs/listdb-server-profile.svg)

### 几个比较有意思的观察：

1. redis-server, listdb-server每秒都可以处理200k次请求
2. redis-server的结果中，`epoll_ctl`和`epoll_wait`耗时都比较多，各7%-8%。但这是在每秒用户程序收发20万次请求的情况下。但正常的程序，很少有每秒这么大量的首发，所以epoll很难成为系统的瓶颈。
3. 仔细观察listdb-server的结果，仅`epoll_ctl`耗时较多，`epoll_ctl`难觅踪影。原因是我做了一个特别的优化，在绝大部分情况下，可以省掉epoll_ctl的调用，并且可以降低latency(稍后会整理一个Pull Request，提给Redis)
4. 没有竞争的锁，开销小。 `listdb-server`图中，有`__pthread_`之类条目，因为`listdb-server`将会是多线程的，每个写会加锁，但这个测试，server是单线程的，所以锁没有竞争。
5. 两个程序中耗时最多的都是 `__write_nocancel`，50%左右，它是`write(2)`，把数据包copy到TCP buffer里
6. `__read_nocancel`也很耗时，占10%左右。它是`read(2)`，从TCP buffer拷贝数据到用户程序的buffer


### redis-server中epoll_ctl耗时

![epoll_ctl](/imgs/redis-epoll_ctl_epoll_wait.jpg)

### listdb-server中epoll_ctl难觅踪影，epoll_wait耗时

![epoll_ctl](/imgs/listdb_epoll_wait.jpg)

