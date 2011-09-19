---
author: feng
date: '2011-09-18 20-21-00'
layout: post
status: publish
title: Async Java HTTP client and DNS resolver
---

I spent some of my spare time writing [Rssminer](http://rssminer.net),
an intelligent RSS reader, I want it to be smart enough to highlight
stories I like, and help me discover stories I may like.

I plan to do it by downloading as many web pages
as possible from the Internet, extract RSS links it contains, download
them, then apply machine learning algorithms on them. It's ambitious.

The first thing need to be solved is an Http client. JDK's
[URLConnection](http://download.oracle.com/javase/1.4.2/docs/api/java/net/URLConnection.html)
is blocking, 20 threads devoted to it, still not fast enough, and
there are some keepalive timer come out the way. The non-blocking
[AsyncHttpClient](https://github.com/sonatype/async-http-client) is
tried, but
[InetAddress.getAllByName(String)](http://download.oracle.com/javase/1.4.2/docs/api/java/net/InetAddress.html#getAllByName(java.lang.String))
 is blocking, slow things down.

In order to be fast and memory efficient, I write my own async HTTP client and
DNS resolver, by using a great library
[netty](http://www.jboss.org/netty), which provides a async socket
framework and HTTP codec.

{% highlight java %}
   // DNS resolver sample usage
   DnsClient client = new DnsClient();
   final DnsResponseFuture f = client.resolve("shenfeng.me");
   f.addListener(new Runnable() {
       public void run() {
            String ip = f.get(); // async
       }
   });
   String ip = f.get(); // blocking
{% endhighlight %}

{% highlight java %}
   // Http client sample usage
   HttpClientConfig config = new HttpClientConfig();
   header = new HashMap<String, Object>();
   HttpClient client = new HttpClient(config);
   URI uri = new URI("http://onycloud.com");
   final HttpResponseFuture future = client.execGet(uri, header);
   resp.addListener(new Runnable() {
       public void run() {
           HttpResponse resp = future.get(); // async
       }
   });
   HttpResponse resp = future.get(); // blocking
{% endhighlight %}

The source code is concise, about 1000 lines of code(about 600 lines
excluding import statements and blank lines), can be found here.
