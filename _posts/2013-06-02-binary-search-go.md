---
author: feng
date: '2013-06-03 08-21-00'
layout: post
status: publish
desc: go，binary search的几种写法，性能对比，以及和buildin的map的性能对比
title: binary array search的几种写法（go）
---

儿童节那天，[对比了python内置的dict和binary array search查找的性能](/python-binary-search-vs-dict.html)，dict大幅领先，有些出乎意料：

1. dict虽然是O(1)，binary search是O(log n)。当n为1M时，log n也仅有20，并且直观上log n前面的常数不大，理论上应该和O(1)差别不是很大
2. 怀疑到python语言，binary array search是由手工python实现，而dict是C实现。python语言比C慢很多

所以，6月2日，用go语言来对比一下，也熟悉一下go语言。
binary search有几种写法：

1. `recusive`: 递归
2. `3-branch`: 迭代，每次迭代进行2次判断 `==`，`<`（或者`>`），三个分支。找到后，立刻退出。最好情况为O(1)
3. `2-branch`: 迭代，每次迭代进行1次判断 `<`（或者`>`），二个分支。最好情况也为O(log n)
4. `library`:  go标准库[`sort.Search`](http://code.google.com/p/go/source/browse/src/pkg/sort/search.go#59)，它是3的通用版本，需要提供一个函数，用来提供`>`或`<`的依据

### 2-branch
3的想法比较有意思，它是先计算出对于要查找的数，如果放入这个已经排好序的数组array，下标是多少。如果在array里，有重复，返回的是则是第一个的下标。最后再对比一下数组里面的数，是否正是需要查找的key

{% highlight go %}
func binary_search2(arr []int, key int) int {
	lo, hi := 0, len(arr)-1
	for lo < hi {
		mid := (lo + hi) / 2
		if arr[mid] < key {
			lo = mid + 1
		} else {
			hi = mid
		}
	}

	if lo == hi && arr[lo] == key {
		return lo
	} else {
		return -(lo + 1)
	}
}
{% endhighlight %}

### library
4的做法很好玩，实现[search.go](http://code.google.com/p/go/source/browse/src/pkg/sort/search.go#59):
{% highlight go %}
func Search(n int, f func(int) bool) int
{% endhighlight %}

go语言没有模版，没有像java那样的Comparator，没有运算符重载，还是静态类型。但这个接口完全不需要这些，并且非常通用（虽有一点性能的损失）

### 3-branch和recusive

1和2是常规解法了。
{% highlight go %}
// recusive
func binary_search0(arr []int, key int) int {
	return binary_search_(arr, 0, len(arr), key)
}

func binary_search_(arr []int, begin, end, key int) int {
	if begin > end {
		return -1
	}
	mid := (begin + end) / 2
	if arr[mid] == key {
		return mid
	} else if arr[mid] > key {
		return binary_search_(arr, begin, mid-1, key)
	} else {
		return binary_search_(arr, mid+1, end, key)
	}
}

// 3-branch
func binary_search(arr []int, key int) int {
	lo, hi := 0, len(arr)-1
	for lo <= hi {
		mid := (lo + hi) / 2
		if arr[mid] == key {
			return mid
		} else if arr[mid] < key { // search upper subarray
			lo = mid + 1
		} else {
			hi = mid - 1
		}
	}
	return -(lo + 1)
}
{% endhighlight %}

### 性能比较
随机生成2k到500k个int，进行1500次查询，纪录下时间。并以内置的map做为对比

![perf](/imgs/binary_search.png)

1. map依然是最快的，但没有像python那样，快得嚣张
2. go标准库的很通用，也是最慢的
3. 随着item数量的增多，2-branch相对与3-branch的优势明显：得益于虽然可能迭代的次数增多（不找到就返回了），但每次2个比较，3个branch ＝> 1个比较2个branch。JDK里面[java.util.Arrays.binarySearch](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/7-b147/java/util/Arrays.java#950)用的是3-branch版本，可能在数量不是很大（少于2k）有优势，或有其它考虑


递归，看上去还行，但易于编写，不易出错。很好

代码[binary_search.go](https://github.com/shenfeng/gocode/blob/master/binary_hash.go)，把结果画成比较好看的柱状图[binary_hash_go.py ](https://github.com/shenfeng/gocode/blob/master/binary_hash_go.py)

ps：这个周末除了和朋友聚聚，为女朋友的侄子采购玩具，剩下就是折腾binary search了。
