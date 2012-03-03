---
author: feng
date: '2012-03-03 09-03-00'
layout: post
status: publish
title: 浅说Javascript的namespace 和 private
categories:
- javascript
---

[Douglas Crockford](http://www.crockford.com/) 说

* [JavaScript: The Wrrrld's Most Misunderstood Programming Language](http://javascript.crockford.com/javascript.html)
* [The World's Most Misunderstood Programming Language Has Become the World's Most Popular Programming Language](http://javascript.crockford.com/popular.html)

我参与的项目用了很多javascript。比如

* [中文版美味书签](http://mei.fm/): 选集查看页面是用javascript渲染的；
  选集创建，js也起了很重要的作用
* [Track](https://trakrapp.com/): 就是一个Javascript App， 很多逻辑都是
  在浏览器完成， 包括routing， 生成HTML......
* [Rssminer](http://rssminer.net): javascript做了更多事情， 比如浏览器端
  生产HTML， 去广告， 实现readability......

依赖Javascript有很多好处，不细说。缺点也有一堆，比如“没有namespace和private”， 他们是模块化和封装的基础，我感觉尤为关键。`package`，`private`是javascript的关键字，但是作用却是：`不小心用作变量名, object的key，程序出错`，仅此而已。`但是上帝在把门关上的时候，留了一个窗户`。用`函数`可以实现他们（从某种意义上说）

{% highlight js %}
// utils.js
(function () {
  //  private. given by closure
  var private_var = 1;

  var helper1 = function () {
  };

  // namespace: YOUR_NS. given by javascript's global object
  window.YOUR_NS = window.YOUR_NS || {};

  window.YOUR_NS.utils = {
    helper1: helper1            // export, public
  };
  // create an anonymous function, execute it immediately
})();
{% endhighlight %}

{% highlight js %}

// app.js
(function () {

  // like java's import, c++'s using namespace
  var utils = window.YOUR_NS.utils;

  // rename
  var utils2 = window.YOUR_NS.utils;

  // direct import, java's import static
  var helper1 = utils.helper1;

  // you app's logic here
  utils.helper1();              // smaple usage
  utils2.helper1();             // smaple usage
  helper1();                    // smaple usage
})();

{% endhighlight %}

{% highlight html %}
<!-- app.html -->
<!-- browser load and execute them in order -->
<script src="utils.js"></script>
<script src="app.js"></script>
{% endhighlight %}

