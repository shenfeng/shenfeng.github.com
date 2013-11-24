---
author: feng
date: '2013-11-24 08-21-00'
layout: post
status: publish
title: go1.2 map的GC性能提升
---

几天前，[线上Golang程序 GC调优一例](http://shenfeng.me/go-gc-optimize-map.html)
介绍了为特定程序优化gc的一个例子，从里面可以看出，go在做map的gc时，性能不太理想（50万的map，在i7-2600s上停顿8ms)

今天时星期天，天气不错！下午出去跑步，上午在家玩一会儿程序。从code.google.com下载了[go1.2rc5](https://code.google.com/p/go/downloads/detail?name=go1.2rc5.linux-amd64.tar.gz&can=2&q=)的包，实际测试这个情况有没有改变。

和上次一样的程序，同一台机器：

{% highlight sh %}
gc32(1): 2+0+0 ms, 61 -> 30 MB 15457 -> 3198 (357463-354265) objects, 0(0) handoff, 0(0) steal, 0/0/0 yields
gc33(1): 2+0+0 ms, 61 -> 30 MB 15470 -> 3198 (369735-366537) objects, 0(0) handoff, 0(0) steal, 0/0/0 yields
gc34(1): 2+0+0 ms, 61 -> 30 MB 15183 -> 3192 (381720-378528) objects, 0(0) handoff, 0(0) steal, 0/0/0 yields
{% endhighlight %}

gc时间由原来的8ms，减少到2ms。

#### GOGCTRACE 参数废弃，替换为GODEBUG

搜索代码树，找不到`GOGCTRACE`字样了，现在由`GODEBUG`环境变量接管。具体代码在[src/pkg/runtime/runtime.c?name=go1.2rc5#386](https://code.google.com/p/go/source/browse/src/pkg/runtime/runtime.c?name=go1.2rc5#386):

{% highlight c %}
static struct {
        int8*   name;
        int32*  value;
} dbgvar[] = {
        {"gctrace", &runtime·debug.gctrace},
        {"schedtrace", &runtime·debug.schedtrace},
        {"scheddetail", &runtime·debug.scheddetail},
};
{% endhighlight %}

{% highlight sh %}
GOGCTRACE=1 go run gc.go  # go1.2以前，GOGCTRACE环境变量控制 详细信息打印
GODEBUG='gctrace=1' go run gc.go  # go1.2rc5 由GODEBUG控制
{% endhighlight %}

### commit 追溯

好奇提升的原因，追了一会儿代码。

go的gc的代码集中在src/pkg/runtime/mgc0.c里。
{% highlight sh %}
# github 上，对golang code的镜像。相比于官方用hg管理，git对于我更友好一点
git clone git@github.com:jnwhiteh/golang.git  
git log src/pkg/runtime/mgc0.c   # 查看 src/pkg/runtime/mgc0.c 的修改记录

{% endhighlight %}

这次map gc性能的提升，可能是Keith Randall的这个[commit](https://github.com/jnwhiteh/golang/commit/8680fa6ebf521b712e57ce3400a1139b3d273de4) 对 [Issue 6119](https://code.google.com/p/go/issues/detail?id=6119)的修复造成的，commit message：

> runtime: record type information for hashtable internal structures.  Remove all hashtable-specific GC code

大量内存数据，造成GC时的长时间停顿，使我头疼。这也是我日常需要面对的：加载大量数据，在有限的时间内（几十ms），进行在线算法计算，返回结果，比如推荐，搜索等。这使我不得不用C++来完成的程序的编写。欣喜的看到这次进步。map应用非常频繁的数据，这次提升，非常有意义。期待go语言gc的继续进步！
