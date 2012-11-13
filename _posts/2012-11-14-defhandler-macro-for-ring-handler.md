---
author: feng
date: '2012-11-14 03-21-00'
layout: post
status: publish
desc:
title: defhandler, a clojure macro aid define ring handler
categories: ['clojure']
---

Currently, I am working at [AVOS Systems's](http://avos.com)
[China team](http://team.mei.fm/).
I am in a group responsible for building [meiweisq](http://meiweisq.com/),
The [Delicious](http://delicious.com/) web service,
but for Chinese users, and using Clojure!

We are using [Ring](https://github.com/ring-clojure/ring) and
[Compojure](https://github.com/weavejester/compojure).
Ring's request/response abstraction is great,
Compojure's `defroutes` family functions get its job done perfectly.
We are just happy using it.

A ring handler is just a plain Clojure function, passing a request map,
return a response map:

{% highlight clojure %}
(defn handler [req]
  ;; return a map of keys [:status :body :headers]
  )
{% endhighlight %}

There are some boilerplate code, in this pattern:
{% highlight clojure %}
;;; get the params
(let [{:keys [id tag]} (:params req)
      limit (min 30 (to-int (or (:limit (:params req)) 15)))
      offset (to-int (or (:offset (:params req)) 0))]
  ;; do work with these params
  )
{% endhighlight %}

They are repeated in the source code. Repeats are bad.
Macro comes to the rescue:

{% highlight clojure %}
(defmacro defhandler [handler bindings & body]
  (let [req (bindings 0)
        bindings (rest bindings)
        ks (map (fn [s] (keyword (name s))) bindings)
        vals (map (fn [k]
                    ;; limit and offset are special, set the default value, guard large
                    ;; limit, convert them to int
                    (cond (= :limit k) `(min 30 (to-int (or (~k (:params ~req)) 15)))
                          (= :offset k) `(to-int (or (~k (:params ~req)) 0))
                          ;; more pattens here, something like
                          ;; (= :uid k) code to get user id
                          :else `(~k (:params ~req)))) ks)]
    `(defn ~handler [~req]
       (let [~@(interleave bindings vals)]
         ~@body))))
{% endhighlight %}

## example usage

{% highlight clojure %}
(defhandler list-feeds [req id limit offset]
  {:status 200
   :body (str "the subscription id: " id
              "limit:" limit
              "offset:" offset)})
{% endhighlight %}

Macroexpand to

{% highlight clojure %}
(defn list-feeds [req]
  (let [id (:id (:params req))
        limit (min 30 (to-int (or (:limit (:params req)) 15)))
        offset (to-int (or (:offset (:params req)) 0))]
    {:status 200
     :body (str "the subscription id: " id
                "limit:" limit
                "offset:" offset)}))
{% endhighlight %}

Boilerplate killed. I am happy now.

By the way, `to-int` source code:

{% highlight clojure %}
(defn to-int [s] (cond
                  (string? s) (Integer/parseInt s)
                  (instance? Integer s) s
                  (instance? Long s) (.intValue ^Long s)
                  :else 0))
{% endhighlight %}
