---
author: feng
date: '2012-6-30 08-21-00'
layout: post
status: publish
desc: Rssminer - an intelligent RSS reader, is public beta now.
title: Rssminer public beta
categories:
- rssminer
---

[Rssminer](http://rssminer.net) is a simple online
[RSS reader](http://en.wikipedia.org/wiki/News_aggregator) written in
Clojure, Java, Javascript.

### Feature

* Not Feature rich, just a central place to reading RSS news.
* Concise UI.
* Read the orignal article, which may include valuable comments.
* Learing from reading history, rank all articles accordingly.
* Fast, Instant full text search. 

### A brief history

1. Initially, I randly pick a project to experiment one page javascript
APP, client side unit testing. A RSS reader is choosen by accident.
2. [Backbone](https://github.com/documentcloud/backbone) is tried, I even
  read [the code](http://documentcloud.github.com/backbone/docs/backbone.html) several times.
  [JavaScript: The Good Parts](http://www.amazon.com/JavaScript-Good-Parts-Douglas-Crockford/dp/0596517742),
  got read several times.
3. [Function](http://en.wikipedia.org/wiki/Functional_programming) and
  [Object literal](https://developer.mozilla.org/en/JavaScript/Guide/Values,_Variables,_and_Literals)
  are the best parts of Javascript. They complement each other. 
  Backbone fail to utilize them efficiently.
4. I did many expriments and thinking. By dropping Backone.js, take my brain as the backone,
   [structure the code](https://github.com/shenfeng/rssminer/tree/master/public/js/rssminer)
   the way I like it, I found a better world.
5. Writing javascript is fun. Writing Clojure is fun. Clojure mixed
   with JAVA is really great. They make me continue doing this project.
6. [Rssminer](http://rssminer.net) is born.

### Performance

I like fast application. In order to be fast,

* The application is carefully designed .
* A [WEB server](https://github.com/shenfeng/http-kit), written from
  scratch to power Rssminer, to achieve low lantency.
* A [HTTP Client](https://github.com/shenfeng/http-kit),
  written from scratch to power the fetcher module.
* A fast [Template](https://github.com/shenfeng/mustache.clj). 
  With it, I can easily pack the minified CSS with the HTML to save a
  network round trip. It bootstrap the application.
* Browser talks to server in JSON. Render HTML in browser side achieves
  even better performance.
* Powered by Redis and MySQL

### Demo
A demo account is created:
[http://rssminer.net/demo](http://rssminer.net/demo).
Feel free to try it. Feedbacks are welcome.

### Open Source
[https://github.com/shenfeng/rssminer](https://github.com/shenfeng/rssminer)
