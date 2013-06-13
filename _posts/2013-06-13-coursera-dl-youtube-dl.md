---
author: feng
date: '2013-06-13 08-21-00'
layout: post
status: publish
desc: 两个实用下载工具：coursera-dl, youtube-dl,下载youtube或者Coursera的视频
title: coursera-dl, youtube-dl,视频下载利器
---

## [coursera-dl](https://github.com/jplehmann/coursera)

Coursera是个好网站，上面有很多课程，[课程列表](https://www.coursera.org/courses)，涵盖多个学科，并且免费。学习充电的好地方。近段时间，我在上面学习：

1. Andrew Ng的 [Machine Learning](https://www.coursera.org/course/ml)，毕业这3年，断断续续在看Machine Learning的东西，但没有系统的学习，Andrew Ng正好系统的讲解，可以弥补一下。
2. Alex Aiken的 [Compilers](https://www.coursera.org/course/compilers), 前段时间，寻找新的工作机会，到阿里面试时，被问到compiler。但我的大学本科是信息与计算科学，学了一大堆数学，编译原理学校给省掉了，自己也没有自学。无奈承认不会，虽他们没有因此拒掉我，但确实不会，念念不忘，Coursera正好有这个课程，正好可以系统学习一下

但家里面的网速，在线看，卡，影响心情。正好遇到coursera-dl，可以完整的把video下载下来，比如Compiler

{% highlight bash %}

# 需要注册Coursera，并且enroll
# 它支持wget，python等下载。wget是久经考研的，相信，比起相信同志，更相信它
coursera-dl -u "<your-email>" -p "<your-password>" -w `which wget` compilers-003

{% endhighlight %}


## [youtube-dl](http://rg3.github.io/youtube-dl/)

Youtube上更是有很多好的视频，比如Google IO等，有的还有1080p源。在线看1080p，需要的是耐心，离线下载能搞定。比如下载 Google I/O 2013: Keynote， [http://www.youtube.com/watch?v=9pmPa_KxsAM](http://www.youtube.com/watch?v=9pmPa_KxsAM):

{% highlight bash %}

# -f 后面的是视频质量，这几个数字分别代表 1080p/720p/480p/360p
#    详细参看http://en.wikipedia.org/wiki/YouTube#Quality_and_codecs
youtube-dl "http://www.youtube.com/watch?v=9pmPa_KxsAM" -f 37/22/35/34

{% endhighlight %}

一个小脚本，下载我感兴趣的Google IO的视频：

{% highlight bash %}

#! /bin/bash

lists=(
    "http://www.youtube.com/watch?v=9pmPa_KxsAM"
    "http://www.youtube.com/watch?v=uzBw6AWCBpU"
    "http://www.youtube.com/watch?v=Jl3-lzlzOJI"
    "http://www.youtube.com/watch?v=NYtB6mlu7vA"
    "http://www.youtube.com/watch?v=qlrKh-L4bqU"
    "http://www.youtube.com/watch?v=XpqyiBR0lJ4"
    "http://www.youtube.com/watch?v=yp8AjMBG87g"
    "http://www.youtube.com/watch?v=lmv1dTnhLH4"
    "http://www.youtube.com/watch?v=LCJAgPkpmR0"
    "http://www.youtube.com/watch?v=GcNNx2zdXN4"
    "http://www.youtube.com/watch?v=_KBHf1EODuk"
    "http://www.youtube.com/watch?v=A5OOJDIrYls"
    "http://www.youtube.com/watch?v=WEBeNZ8khS4"
    "http://www.youtube.com/watch?v=x6qe_kVaBpg"
);

for l in "${lists[@]}"
do
    echo "downloading $l"
    # http://en.wikipedia.org/wiki/YouTube#Quality_and_codecs
    youtube-dl "$l" -f 37/22/35/34
done

{% endhighlight %}

端午节时，三天假期，五台山远足。脚本跑着。回来时，已经搞定。
