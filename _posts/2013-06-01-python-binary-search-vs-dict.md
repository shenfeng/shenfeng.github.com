---
author: feng
date: '2013-06-01 08-21-00'
layout: post
status: publish
desc: python语言，binary search对比dict，dict比binary search快不少
title: python语言，binary search和dict对比
---

今天是国际儿童节。
跟我没有什么关系。
祝他们都能开心，快乐的成长。

dict可以用来判断一个元素是不是在集合里面，binary search也可以。一直想对比一下他们的性能：

1. 查找，hash是O(1), binary search是O(logn)
2. 由于array内存非常紧凑，可能内存访问会少一些，它还更省内存
3. 我写过几个程序，array的数据是通过mmap得到，是别的程序排好序，写入文件的
4. 有些语言，如C，标准库没有dict（map）的实现
5. 个数从2k到200k，1500次查询，750个能查到，另外750个可能查到

## 测试代码

测试代码在[github](https://github.com/shenfeng/gocode/blob/master/binary_hash.py)。其中binary search的实现为：

{% highlight python %}
def binary_search(arr, key):
    lo, hi = 0, (len(arr) - 1)
    while lo <= hi:
        mid = (lo + hi) >> 1
        k = arr[mid]
        if k == key:
            return mid
        elif k < key:
            lo = mid + 1 # search upper subarray
        else:
            hi = mid - 1 # search lower subarray
            # (-(insertion point) - 1)
    return -(lo + 1)
{% endhighlight %}

随机生成测试数据的代码
{% highlight python %}
COUNT = 1000 * 200

# key is string
key_str = {}
for i in range(COUNT):
    key_str[random_string(random.randint(2, 15))] = True

# key is int
key_int = {}
for i in range(COUNT):
    key_int[random.randint(0, 1 << 32)] = True
{% endhighlight %}


#### 长度为2到15的随机字符串

![string](/imgs/py_dict_bs/string.png)

#### 随机int，array为list

![string](/imgs/py_dict_bs/int.png)

#### 随机int，用[array module](http://docs.python.org/3/library/array)

![string](/imgs/py_dict_bs/int_array.png)

## 结论

1. dict完胜binary search，当数据仅有2000个时，依然
2. 随着item个数的增加，dict性能几乎不变：O(1)不是说着玩的
3. array没有list效率高，倒是出乎意料。文档说它很快，binary search时，是随机访问
4. binary search慢，有可能是python慢，dict是C实现的，并且是使劲优化过的
5. int和string没有明显的性能差别
6. 需要测试一下其它语言，如go，java，c。再找时间了
