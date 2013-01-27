---
author: feng
date: '2013-1-27 22-27-00'
layout: post
status: publish
title: Add async (long polling) to ring web application
categories: ['http-kit', 'clojure']
---

*This is the second post of introducing [http-kit](http://http-kit.org), the first one [A new Clojure HTTP client, concurrent made easy by asynchronous and promise](http://shenfeng.me/async-clojure-http-client.html)*

### [Ring](https://github.com/ring-clojure/ring), Clojure's way of abstrating HTTP

> Ring is defined in terms of handlers, middleware, adapters, requests maps, and response maps

{% highlight clojure %}
(defn handler [request-req] ;; accept a reuqest-map, return a response-map
  response-map)

;; request-map
{:uri "/index"
 :query-string "a=b"
 :request-method :get
 :headers {"user-agent" "... Chrome/23.0.1271.40 Safari/537.11"
           "cookie" "...."
           "accept-charset" "UTF-8,*;q=0.5"}
 ...}

;; response-map
{:status 200
 :headers {"Content-Type" "text/plain"}
 :body "Hello world from ring"
 :session ...}

;; adapter
(run-server handler {..options..}}
{% endhighlight %}

Request passed in, response expected.
There is no support of async(long polling) in ring's Spec.
Most of the time, You don't need async.
When async is needed, like server need to push data to client in realtime, [http-kit](http://http-kit.org) offers an option:

### Async with `async-response`

http-kit is a HTTP server/client written from scrach for Clojure.
The server is a standard ring adapter with async and websocket support, a drop in replacement of ring-jetty-adapter.

{% highlight clojure %}
[http-kit "2.0-rc1"] ;; add to project.clj
{% endhighlight %}

Only a single interface needed to async support: `async-response`

{% highlight clojure %}
(defn handler [req-map]
  (async-response respond
    (future (respond {:status 200
                      :headers {"Content-Type" "text/plain"}
                      :body "async hello world"}))))
(run-server handler {:port 8000}}
{% endhighlight %}

1. `async-response` is macro, it gives you `respond`
2. `respond` is a function, cann be saved, used to send response to client, on any thread, at any time.

Credit for this flexible API goes to [Peter Taoussanis](https://github.com/ptaoussanis).

Code snippet:

{% highlight clojure %}
(def clients (atom {}))

(async-response respond
  ;; save it
  (swap! clients assoc respond (now-seconds)))

(on-fancy-event (fn [e]
                  (doseq [client (keys @clients)]
                    (client {:status 200
                             :body (str "Fancy event!" e)})
                    (swap! clients dissoc client))))
{% endhighlight %}

### What's goes on under the hook?

* `request` received by http-kit, goes though all request-level middlewares, get *modified*
* `async-response` return a standard ring repsonse, except the body is IListenableFuture. http-kit understand the body, holds on the request.
* A callback is registed on IListenableFuture, waiting `respond` to return the *actual* response
* `respond` send to *actual* response to client (response-level middlewares are not applied), client's request returns

### Efficiency

Keep a connection costs nothing but a few kilobytes of RAM. http-kit runs happily with ~128M heap while ~10K high traffic concurrent connections.

{% highlight sh %}
# http-kit runs happily with 128M heap (-Xms128m -Xmx128m), ab confirms that
ab -n 500000 -c 10000 -k http://127.0.0.1:8080/
{% endhighlight %}

### Open source

http-kit's website: [http-kit.org](http://http-kit.org)

Source code is hosted on github: [github.com/http-kit/http-kit](https://github.com/http-kit/http-kit)

To suggest a feature, report a bug, or general discussion: [github.com/http-kit/http-kit/issues](https://github.com/http-kit/http-kit/issues)
