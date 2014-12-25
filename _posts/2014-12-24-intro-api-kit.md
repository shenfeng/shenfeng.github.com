---
author: feng
date: '2014-12-24 08-21-00'
layout: post
status: publish
title:  自动生成Android，iOS的REST API访问代码
---

**我们正在招聘，需要Android和iOS工程师的加入，和我一组，开发[看准网](http://www.kanzhun.com) 的App，为四亿职场人服务。
请发简历 到 shenfeng at kanzhun dot com，请加入我们**

上个月，介绍了[生成Java JDBC访问代码，从SQL文件](/java-jdbc-generate-boilerplate.html)，后面应用到我负责的几个实际的项目，效果不错，
程序清晰了很多，也更好维护。

用类似的思路，我做了一个新的工具: api-kit，移动开发从中受益。

### 移动开发：Android，iOS，RESTful API

移动开发，下面几个环节免不了：

1. 负责Server端开发的同学，写API，写文档
2. 负责Android的同学，根据文档，以及和server端同学的交流，实现和服务器的通信代码
3. 负责iOS的同学，根据文档，以及和server端同学的交流，实现和服务器的通信代码

这里面有一些重复： 负责server的同学开发API后，写文档是把同样的信息，表达了两次，一次是在code里面，一次在文档。
客户端同学实现各自平台的访问代码，是重复，因为这部分信息是server端的code（或者文档）的一次翻译（跨语言，跨平台）。

1. 信息在这个过程中是无创造的复制（或者翻译，即没有引入更多的信息）
2. 不能保证复制的正确性，比如文档错误，客户端写code引入bug，等
3. 复制的成本较高：沟通成本，写code的成本
4. 如果API重构，这个改动，需要在其它几个地方replay

这几天，我写了一个程序，api-kit，通过自动生成代码的方式，解决这里面的大部分重复，使信息的流动更高效和快捷。

### 例子

假设API描述是这样的:

{% highlight cpp %}
// 这是api-kit的输入，输出是各个平台的code：Android, iOS, Server端
struct Book {
    i32 id
    string title
    string isbn
    float32 price
    string description
}

@url(/books/newest)
@get
func list<Book> getNewest(i32 limit, i32 offset);

@url(/books/search)
@get
func list<Book> searchBook(string q, i32 limit, i32 offset);

{% endhighlight %}

**API描述是api-kit的输入，输出是各个平台（Server，Android, iOS）的code。**

### 生成的Server端code

1. IHandler interface
2. Book类
3. hook这个interface到servlet的支持代码（参数解析，绑定，dispatch）

{% highlight java %}
public interface IHandler {
    // Called before every function. Use cases: setup context, authentication return false to abort further execution.
    public boolean before(Context context);

    // Called after every function. One use case is logging
    public void after(Context context);


    // GET /api/books/newest
    public List<Book> GetNewest(Context context, int limit, int offset);

    // GET /api/books/search
    public List<Book> SearchBook(Context context, String q, int limit, int offset);
}

服务器端的工作简化为实现这个Interface

{% endhighlight %}

### 生成的Android端code

生成的code，处理url拼接，返回值解析，暴露给程序员的是函数：

![2](imgs/apikit/android.png)

零依赖，能有效的减少apk的体积。在compiler的帮助下，做到类型安全，服务器重构后，这边会编译出错，refactor会方便一些。


### 生成的iOS的code

iOS显然是需要支持的。先花了一天时间，学习swift，并完成swift的code生成的生成。 在和公司的iOS工程师沟通时，他们在用objective-c，
于是又多花了一天时间看oc，并生成oc的code，语法有点不太习惯，但还是搞定了。

![2](imgs/apikit/ios.png)

也是零依赖的，能有效的减少app的体积。由于oc支持类型安全的dict和arr挺困难，于是，通过注释帮助一点。

![2](imgs/apikit/os_type.png)

### Batch，打包多个请求

由于移动的特殊性（latency），一个页面一般需要请求多次，才能render。请求多用异步，多个异步的嵌套挺不方便，
如果能合并这些请求，移动端的开发会更方便，也会提高性能。api-kit透明的实现了这一点。

{% highlight cpp %}
ns com.kanzhun.api

struct Book {
    i32 id
    string title
    string isbn
    float32 price
    string description
}

struct NewestReq {
    i32 limit
    i32 offset
}

struct SearchBookReq {
    string q
    i32 limit
    i32 offset
}

@url(/books/newest)
@get
func list<Book> GetNewest(NewestReq req); // batch要求参数的个数仅为一个

@url(/books/search)
@get
func list<Book> SearchBook(SearchBookReq req);

// `batch` 为关键字
// GetNewest, SearchBook 都是函数名
// batch请求，和服务端是一次交互，客户端发送一次请求给服务端

@url(/batch)
batch Batch(GetNewest, SearchBook)

{% endhighlight %}

生成的code

1. 在服务器端透明的处理掉Batch请求
2. 在客户端生成名为Batch的函数，接受GetNewest的req，SearchBook的req，返回BatchResp

![2](imgs/apikit/batch.png)

![2](imgs/apikit/batch_ios.png)


### 现在的状态

1. Android，iOS，服务端（servlet）已经完成，已测试。生成的code和手写的代码一样，高效简洁。
2. 除去Batch，没有任何的私有协议。这里采用的是大家熟悉的HTTP，JSON，RESTful。 事实上，api-kit也可以用来生成已有的restful api的客户端code。
3. Batch用了一点私有协议： 请求打包，post给服务器，服务器拆开，取出一个一个的请求，挨个调用，打包每个函数的返回值，返回客户端。Batch的服务端逻辑，是自动生成的
4. Api的定义文件，是个很好的文档。api-kit把文档，翻译成了可以执行的代码。代码也是文档。这会节约团队之间的交流成本，
节约出来的时间，可以干更有意思的事情，比如晒晒太阳，喝喝咖啡，和奶奶聊聊天，听她讲故事。
5. Rest API的url endpoint是什么，对于客户端来说，变成了实现细节。客户端变成仅关心 函数名，参数，返回值，而这些，IDE会给我们很好的帮助。

### 接下来的工作

1. 可能我会想办法做到透明的cache（生成cache的code），
2. 支持其它语言，比如生成ajax调用的js code，用于支持网站开发，Python的客户端（开发时的黑盒子测试）


**我们正在招聘，需要Android和iOS工程师的加入，和我一组，开发[看准网](http://www.kanzhun.com) 的App，为四亿职场人服务。
请发简历 到 shenfeng at kanzhun dot com，请加入我们**
