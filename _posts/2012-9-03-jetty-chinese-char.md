---
author: feng
date: '2012-9-03 09-21-00'
layout: post
status: publish
desc: jetty在处理http response header中有中文字符时，会出现乱码，原因是hardcode了ISO-8859-1，浏览器收到是问号
title: jetty处理 http response header中文字符，调试笔记
categories:
- web
---

这段时间，我和几个同事在做另外一个`美味书签`，一个没有选辑，专注收藏的服务，和有选辑的美味书签并存。

大家讨论，觉得url应该是尽量清晰简单。比如`张三`用户登陆，它的url是`/u/张三`，之类的。能让高级用户直接输入url。url是可读的。

我们的程序用clojure开发。用ring，compojure, jetty。

我遇到了一个问题，在jetty返回给浏览器的http header中， 中文成了乱码。

场景如下：

1. 张三到达我们网站，他在美味书签的用户名也是张三，输入的网址是landing page的 url： http://meiweisq.com/
2. 程序发现，此用户已经登陆过，redirect到他的首页 `/u/张三`
3. 浏览器看见的却是 /u/??

在第2和第3步之间出问题了。jetty在encode
    {% highlight clojure %}
    {
     :status 302
     :headers {"Location" "/u/张三"}
     :body nil
     }
    {% endhighlight %}
时出现乱码。尝试各种方法无果。

无奈之下换上我手工写的一个[ring adapter](https://github.com/shenfeng/http-kit)， 发现也出现乱码，赶快修改了encode header相关代码， 从`ASCII`改为`UTF8`，问题解决。

但这只是一个临时的解决方法。

下班回家后，还在琢磨这件事， download了jetty的代码：

{% highlight sh %}
git clone git://github.com/eclipse/jetty.project.git
{% endhighlight %}

checkout 我们用的相应版本7.6.1.v20120215。

经过一番查找，调试，发现了问题所在， jetty在encode http response
header时，用了hardcode的 `ISO-8859-1`

这段代码是用来encode http header的:

{% highlight java %}
// ./jetty-http/src/main/java/org/eclipse/jetty/http/HttpFields.java
// 322行到347行
  private Buffer convertValue(String value)
    {
        Buffer buffer = __cache.get(value);
        if (buffer!=null)
            return buffer;

        try
        {
            buffer = new ByteArrayBuffer(value,StringUtil.__ISO_8859_1);

            if (__cacheSize>0)
            {
                if (__cache.size()>__cacheSize)
                    __cache.clear();
                Buffer b=__cache.putIfAbsent(value,buffer);
                if (b!=null)
                    buffer=b;
            }

            return buffer;
        }
        catch (UnsupportedEncodingException e)
        {
            throw new RuntimeException(e);
        }
    }
{% endhighlight %}

`StringUtil.__ISO_8859_1`的定义

{% highlight java %}
// ./jetty-util/src/main/java/org/eclipse/jetty/util/StringUtil.java

    public static final String __ISO_8859_1="ISO-8859-1";
    public final static String __UTF8="UTF-8";
    public final static String __UTF8Alt="UTF8";
    public final static String __UTF16="UTF-16";
{% endhighlight %}

### 解决办法

{% highlight clojure %}
{
:status 302
:headers {"Location" (URLEncoder/encode "/u/张三")}
:body nil
}
{% endhighlight %}

后面想到可以用另外的方式绕过jetty编码http header的问题：

* 返回一个网页，网页里面用js

{% highlight js %}
location.href="/u/张三"
{% endhighlight %}

* [Meta refresh](http://en.wikipedia.org/wiki/Meta_refresh)

{% highlight html %}
<meta http-equiv="refresh" content="0;URL='/u/张三'">
{% endhighlight %}


