---
author: feng
date: '2013-03-19 08-21-00'
layout: post
status: publish
desc: 浅议如何实现线程池中部分task有序执行
title: 浅议如何高效实现线程池中部分task有序执行
---

线程池接受task，并执行他们。
{% highlight sh %}
T1
T2 *
T3
T4
T5 *
T6
{% endhighlight %}

不做特殊处理，线程池不能保证T1-T6的执行顺序，有可能它们是并发的。但有特殊要求，比如需要T2和T5需要顺序执行，应该怎么实现呢？

问题来源于，在写[http-kit](http://http-kit.org)的时候，我对它加入了WebSocket协议的支持。
处理WebSocket时，由于一个client可能发多个message给服务器，服务器应该需要保证它们的时序：先到现被处理，后到的要等待前面的被处理后才能被处理。

http-kit的线程模型是一个异步的IO线程，负责接受客户端的连接，读取并解析协议为request，扔到一个线程池，由它们计算response，然后IO线程负责把response写到客户端的TCP socket buffer。
http-kit需要处理HTTP和WebSocket协议，HTTP协议是单一的request => response，没有顺序问题。

问题变为：**如何高效实现线程池中部分task有序执行**

简单并高效的代码总是很招人喜欢，我甚至愿意用一点点效率来换取简单。http-kit用3000来行代码，从零高效实现HTTP server，支持基于HTTP长连服务器推送和Websocket，异步的HTTP client，Timer，并用Clojure暴露一个漂亮的API。实际经验告诉我，简单可能会换来性能，比如http-kit具有和Nginx相似的性能特点，[这里](https://github.com/ptaoussanis/clojure-web-server-benchmarks)有一南非朋友做的测试，我还很无聊的做过一个[C600K的试验](http://shenfeng.me/600k-concurrent-connection-http-kit.html)

最先的考虑是需要在线程池上做文章，于是就试着去实现一个特殊的线程池：多个线程，共享一个全局queue，每个还有一个单独的queue，通过单独的queue保证顺序。因为2个queue，需要解决Task被饿死的情况。
试着写了[几个实现](https://github.com/http-kit/http-kit/tree/protocol-api/test/java/org/httpkit/server)，总觉不太对，感觉自己能力有限，试着发了封邮件到Concurrency-interest@cs.oswego.edu，那里面有很多这方面的专家，比如大名鼎鼎的j.u.c的作者Doug Lea，希望能得到点指点。第二天，有几个人给了回复，其中Jeff Hain的回复让我茅塞顿开：


> It seems to me you don't need a particular thread pool to have some of your
> tasks ordered, if you can keep a reference to last submitted runnable for each
> ordered sequence of runnables (for example using a map in IO thread).

> I'm thinking about using a specific "LinkingRunnable", containing
> an "impl" Runnable and an AtomicReference to a "next" Runnable.

> When LinkingRunnable.run() is called, it first runs impl.run(), and then
> does CAS(null,this) (using "this" as a tombstone): if CAS fails, that means
> someone added a runnable to run next, and it runs it.

> When subsequent task arrives, you try to "enqueue" it doing CAS(null,next):
> If CAS succeeds, that means it'll be executed just after previous task
> (and in same thread), and if it fails that means that the previous task
> already completed, so you can just submit your next task to the pool.

> You can also test "get() == null" before doing CASes, which should
> make them rare, and reduce the usual overhead to a volatile read.

我立刻做了实现：

{% highlight java %}

class LinkingRunnable implements Runnable {
    private final Runnable impl;
    AtomicReference<LinkingRunnable> next = new AtomicReference<LinkingRunnable>(null);

    public LinkingRunnable(Runnable r) {
        this.impl = r;
    }

    public void run() {
        impl.run();
        if (!next.compareAndSet(null, this)) { // has more job to run
            next.get().run();
        }
    }
}

WSFrameHandler task = new WSFrameHandler(channel, frame);
LinkingRunnable job = new LinkingRunnable(task);
// channel.serialTask 为一个指针，先取出原来的，并更新
LinkingRunnable old = channel.serialTask;
channel.serialTask = job;

if (old == null) { // No previous job
    execs.submit(job);
} else {
    if (old.next.compareAndSet(null, job)) {
        // successfully append to previous task
    } else {
        // previous message is handled, order is guaranteed.
        execs.submit(job);
    }
}

{% endhighlight %}

更详细的代码，以及上下文，请参看[RingHandler.java](https://github.com/http-kit/http-kit/blob/protocol-api/src/java/org/httpkit/server/RingHandler.java)

简单高效的实现了局部有序，非常喜欢，符合我对好代码的要求：简单，API友好，高效
