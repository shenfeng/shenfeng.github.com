---
author: feng
date: '2013-11-13 08-21-00'
layout: post
status: publish
title: 线上Golang程序 GC调优一例
---

Golang
是一个很有意思的语言，第一次看它介绍时，就很喜欢。半年前加入美团，有机会用它写了几个线上程序。其中一个程序Router，每天需要转发几千万的请求。由于需要根据请求内容决定route路径，它需要加载几十万deal（美团单）的信息到内存供查询。问题来了，用map装的几十万数据让gc很辛苦。

### Deal数据

{% highlight go %}
// Deal的定义
type DealTiny struct {
	Dealid    int32
	Classid   int32
	Mttypeid  int32
	Bizacctid int32
	Isonline  bool
	Geocnt    int32
}
{% endhighlight %}

### gc停顿
用go写一个简单的Web程序，设置`GOGCTRACE`环境变量为1后启动程序，用[wrk](https://github.com/wg/wrk)压力测试，观察控制台打出的gc停顿时间。

{% highlight sh %}
GOGCTRACE=1 go run gc.go  # 设置环境变量，go gc时会打印详细信息

wrk http://localhost:8080/ -d 10s  # 压力测试，发送大量请求，让程序“忙”起来，触发gc
{% endhighlight %}

测试程序主要部分code：

{% highlight go %}

func main() {
	const SIZE = 500000 // 50万
	m := make(map[int32]DealTiny, SIZE)
	for i := 0; i < SIZE; i++ { // 把数据放进内存
		m[rand.Int31()] = DealTiny{}
	}
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		// 模拟内存分配，做一些计算
		n := rand.Intn(4096) + 1024
		buffer := make([]int, n)
		for i := 0; i < n; i++ {
			buffer[i] = rand.Intn(1024)
		}
		c := 0
		for i := 0; i < n; i++ {
			if buffer[i] > 512 {
				c += 1
			}
		}
		fmt.Fprintf(w, "n: %d, more than 512 count: %d", n, c)
	})
	log.Fatal(http.ListenAndServe(":8080", nil))
}

{% endhighlight %}

程序在控制台的部分输出

{% highlight sh %}
# go 1.1.1; Linux 3.2.0; CPU Intel(R) Core(TM) i7-2600 CPU 3.40GHz

gc83(1): 8+0+0 ms, 62 -> 31 MB 19455 -> 3211 (1291202-1287991) objects, 0(0) handoff, 0(0) steal, 0/0/0 yields
gc84(1): 8+0+0 ms, 62 -> 31 MB 19087 -> 3213 (1307079-1303866) objects, 0(0) handoff, 0(0) steal, 0/0/0 yields
gc85(1): 8+0+0 ms, 62 -> 31 MB 18935 -> 3212 (1322802-1319590) objects, 0(0) handoff, 0(0) steal, 0/0/0 yields
{% endhighlight %}

gc停顿时间为8ms，且线上CPU比测试机主频低，且是虚拟机，停顿时间比8ms长一些。这么长的停顿时间，显然是不能接受的。需要想办法优化。

查看go的代码 [src/pkg/runtime/mgc0.c#985](https://code.google.com/p/go/source/browse/src/pkg/runtime/mgc0.c?name=release-branch.go1.1#985)发现，gc时，需要一个一个的扫描map的key和value，自然是相当贵的。


go没有像jvm那样多的可以调整的参数，并且不是分代回收。优化gc的方式仅仅只能是通过优化程序。但go有一个优势：有真正的array（而仅仅是an array of referece）。go的gc算法是mark
and
sweep，array对此是友好的：整个array一次性被处理。可以用一个array用open addressing的方式实现map，以此优化gc（也会减少内存的使用，后面可以看到）

{% highlight go %}

// DealMap 为array backend hash table
dm := NewDealMap(SIZE)
for i := 0; i < SIZE; i++ {
    dm.Put(DealTiny{Dealid: rand.Int31()})
}

{% endhighlight %}

此次，gc日志为

{% highlight sh %}
gc80(1): 0+0+0 ms, 25 -> 12 MB 7235 -> 803 (507340-506537) objects, 0(0) handoff, 0(0) steal, 0/0/0 yields
gc81(1): 0+0+0 ms, 25 -> 12 MB 7184 -> 803 (513722-512919) objects, 0(0) handoff, 0(0) steal, 0/0/0 yields
gc82(1): 0+0+0 ms, 25 -> 12 MB 7340 -> 803 (520260-519457) objects, 0(0) handoff, 0(0) steal, 0/0/0 yields
{% endhighlight %}

可以看出，gc回收非常迅速（0ms），并且内存使用也由原来gc后的31M 减少到12M。优化效果是很明显的。


### DealMap的实现

{% highlight go %}
type DealMap struct {
    table   []DealTiny
    buckets int
    size    int
}

// round 到最近的2的倍数
func minBuckets(v int) int {
    v--
    v |= v >> 1
    v |= v >> 2
    v |= v >> 4
    v |= v >> 8
    v |= v >> 16
    v++
    return v
}

func hashInt32(x int) int {
    x = ((x >> 16) ^ x) * 0x45d9f3b
    x = ((x >> 16) ^ x) * 0x45d9f3b
    x = ((x >> 16) ^ x)
    return x
}

func NewDealMap(maxsize int) *DealMap {
    buckets := minBuckets(maxsize)
    return &DealMap{size: 0, buckets: buckets, table: make([]DealTiny, buckets)}
}

// TODO rehash策略
func (m *DealMap) Put(d DealTiny) {
    num_probes, bucket_count_minus_one := 0, m.buckets-1
    bucknum := hashInt32(int(d.Dealid)) & bucket_count_minus_one
    for {
        if m.table[bucknum].Dealid == 0 { // insert, 不支持放入ID为0的Deal
            m.size += 1
            m.table[bucknum] = d
            return
        }
        if m.table[bucknum].Dealid == d.Dealid { // update
            m.table[bucknum] = d
            return
        }
        num_probes += 1 // Open addressing with Linear probing 
        bucknum = (bucknum + num_probes) & bucket_count_minus_one
    }
}

func (m *DealMap) Get(id int32) (DealTiny, bool) {
    num_probes, bucket_count_minus_one := 0, m.buckets-1
    bucknum := hashInt32(int(id)) & bucket_count_minus_one
    for {
        if m.table[bucknum].Dealid == id {
            return m.table[bucknum], true
        }
        if m.table[bucknum].Dealid == 0 {
            return m.table[bucknum], false
        }
        num_probes += 1
        bucknum = (bucknum + num_probes) & bucket_count_minus_one
    }
}

{% endhighlight %}
