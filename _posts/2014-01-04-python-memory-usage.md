---
author: feng
date: '2014-01-04 08-21-00'
layout: post
status: publish
title:  小探Python内存使用overhead
---

##  内存使用10倍以上，出乎预料

周五在公司时，继续处理推荐相关的code，做**物品**之间的相似度矩阵计算：通过<用户，物品>矩阵，可以计算出物品(item)两两之间的相似度，生成相似度矩阵，它是协同过滤的基础数据。对每个物品，仅保留与它最相近的120个物品，以及相似度。程序用C++写的，加上用手工优化的hashmap，多核心同时计算，跑满8个CPU，2分钟左右可以算完600万左右的item的相似度计算。

算完后，相似度关系存为binary的文件（为了快速load）。binary仅对机器友好，对人不友好。得想办法让人可以方便得查看。打算用tornado做一个web界面，可以随便点击查看，这样也给展示更详细得信息提供了可能性：物品以ID表示，如团购项目ID。web工具，可以通过查询数据库，显示团购项目的详细信息，如标题，价格，品类，销量，商家，地址等。这工具能方便我了解数据，培养sense，以及算法排查，领导查看也方便（相比于给领导一个binary文件，似乎给一个可以点击的URL靠谱一些）。

由于相似度文件1G多一点(binary)，C++ load到内存后，RAM在1.5G内。知道python耗内存，但开发机有16G内存，按照Python比C++多耗10倍计算，程序也能跑下来。基于这个考虑，迅速coding。C++load时，耗时在2s内，python慢悠悠的，30s还没有见停。浏览了会儿Hacker News。感觉机器卡：机器在辛苦的swap，Python进程已耗掉了12G+内存。果断`kill -9`。

Python耗内存，早有心理准备，但一个数量级的内存跑程序还困难，还是出乎了意料。

## C++处理
相似度在文件和内存都是map的形式：

{% highlight sh %}

# item_id 以及和它相似的 items，以及相似度score
item_id => (item_id, score)+

{% endhighlight %}
	
(item_id, score)在C++里面是一个struct：
{% highlight cpp %}
struct IdScore {
    int id;
    float score;
};
# 在内存中，耗掉8bytes
sizeof(IdScore) => 8
{% endhighlight %}

C++通过mmap文件，通过强制类型转化，把文件中bytes转成了IdScore array。简单迅速。

## 通过sys.getsizeof查看对象大小

Python没有那么方便，需要通过`struct.unpack` 成tuple。

相对与C和C++里面的`sizeof`, Python里可以则是`sys.getsizeof`。

{% highlight python %}
import sys
sys.getsizeof((1, 1.0))  # 72
{% endhighlight %}

有些出乎意料，tuple算是很节约的了，也需要72 字节，是C++的9倍。

{% highlight python %}

print sys.getsizeof([1, 1.0]) # 88 list。 list比tuple "贵" 一些
print sys.getsizeof(1) # 24 int。int不便宜。
print sys.getsizeof("") # 37 empty string。空字符串也要37 bytes，不便宜。
print sys.getsizeof("abc") # 40 string with 3 chars

class IdScore(object):
    def __init__(self, id, score):
        self.id = id
        self.score = score

# 64。貌似比tuple少一点。那是假象，每个IdScore对象还包含一个__dict__
# getsizeof 算 __dict__8个字节（一指针），但__dict__是独享，需要加进去
# 可以通过__slots__ 节约 __dict__开销
print sys.getsizeof(IdScore(1, 1.0))  
{% endhighlight %}

数字`1`耗掉了24字节，空字符串耗掉37字节。


## 翻阅cpython源码

Python 的 int 实际上是一个 struct `PyIntObject` ，[intobject.h#L23](https://github.com/python/cpython/blob/2.7/Include/intobject.h#L23)

{% highlight cpp %}

typedef struct {
    PyObject_HEAD
    long ob_ival;
} PyIntObject;

/* PyObject_HEAD 的定义，在 https://github.com/python/cpython/blob/2.7/Include/object.h#L65  */

/* Define pointers to support a doubly-linked list of all live heap objects. */
#define _PyObject_HEAD_EXTRA            \
    struct _object *_ob_next;           \
    struct _object *_ob_prev;

{% endhighlight %}

两个指针，一个long，难怪有24bytes。

Python的其它对象也有类似的struct结构（至少有`PyObject_HEAD`），不便宜。

## 后记

虽然Python多用了很多珍贵的内存，但开发迅速，能节约很多宝贵的时间（用少量的money换时间，早实践了。2年前为自己配的台式机，果断上i7 2600，16G。今年初入的Macbook Pro retina也是16G的配置，貌似是非定制化的顶配）

能快速prototype，验证想法是非常重要的。

一般情况下，程序的优化是可以通过重新设计数据结构达到的。这也不例外。想到了一个很简单的优化，让python也能用不到2G内存，处理这堆数据了：

1. mmap 数据文件到内存为array
2. 建立Index：map，存储 item_it => array的下标（和老版本不同，老版本是把数据 `unpack` 到内存了。
3. 查询时，通过map查到下标，再通过`struct.unpack` 得到和它相似的其它 items，以及score