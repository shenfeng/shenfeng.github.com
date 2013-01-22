---
author: feng
date: '2013-1-22 22-21-00'
layout: post
status: publish
title: http-watcher，用go为前端同学写的web服务器
categories: ['go', 'tool']
---

前端同学的开发流程：

![前端开发流程](/imgs/front-end-cycle.png)

如果有这样一个**工具**：监控代码目录，发现代码修改后，**自动**刷新浏览器，流程可减少一步：

![省去刷新](/imgs/front-end-cycle-4.png)

减少一小步，效率提高一大步。因为在大多数情况下，另外两步也可以省掉，就变成了更改代码后，省掉窗口切换，用眼睛扫一下就可以。眼睛的速度很快的哦。**专注于代码和思考**

![省去切换](/imgs/front-end-cycle-2.png)

### [http-watcher](https://github.com/shenfeng/http-watcher)，为前端同学量身打造

俺做前端开发好几年了，切换窗口了好几年。

最近在做 [美味书签] (http://meiweisq.com)，采用了一个支持HTTP长连(服务器可以实时push消息给客户端)的[服务器](https://github.com/shenfeng/http-kit)，
试着加入一些代码，监控代码变化，发现变化，立刻刷新浏览器。用了后，还是比较实用。设计师使用后，也觉得不错。

元旦回家后，各方面原因，加上有些空闲时间，就用google发布的一个比较新的语言go，实现了一个小工具[http-watcher](https://github.com/shenfeng/http-watcher)，也当做学习这们语言。

###用法：静态代码（设计师常用）

http-watcher = HTTP文件服务器 ＋ 文件监控器

{% highlight sh %}

cd {code-directory} && http-watcher -port=8000
# 详细帮助 http-watcher -h
# 打开浏览器，访问http://127.0.0.1:8000。同事们也可通过你的IP，实时查看

{% endhighlight %}

###用法：动态程序(Java, go, Clojure, Python, etc)

http-watcher = HTTP反向代理 ＋ 文件监控器

{% highlight sh %}

http-watcher -port=8000 －root={代码目录} -proxy={动态程序端口}
# 详细帮助 http-watcher -h
# 打开浏览器，访问http://127.0.0.1:8000。同事们也可通过你的IP，实时查看

{% endhighlight %}

### 开源
代码托管于github上面 [http-watcher](https://github.com/shenfeng/http-watcher)

可checkout 代码，自己编译 `go build`，也可以下载

### 下载地址
1. [Linux](http://http-watcher.googlecode.com/files/http-watcher-osx)
2. [Mac OS X](http://http-watcher.googlecode.com/files/http-watcher-linux)
3. [Windows](http://http-watcher.googlecode.com/files/http-watcher-win.exe)
