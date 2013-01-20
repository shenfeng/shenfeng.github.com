---
author: feng
date: '2013-1-21 22-21-00'
layout: post
status: publish
title: A new Clojure HTTP client, concurrent made easy by asynchronous and promise
categories: ['http-kit', 'clojure']
---

[http-kit](https://github.com/shenfeng/http-kit) is a clean and compact event driven Clojure HTTP toolkit, writen from scratch for high performance Clojure applications. It consists
1. Ring compatible adapter with async and websocket support
2. Fast, asynchronous HTTP client, API modeled after `clj-http`

In 2.0, by leverageing `promise`, `deref` mechanism, and `callback`, we redesigned the client's API. Now, the asynchronous API is fast, flexible, and very easy to use.
Thanks [Peter Taoussanis](https://github.com/ptaoussanis) and [Max Penet](https://github.com/mpenet) for their great contributions.

After months's tweaking and testing, it has finally reached the 2.0 release candidate phrase.

<!-- The API is carefully designed. -->

<!-- After a few months testing -->

## Why another Clojure HTTP client
Compare with the very good Clojure HTTP client [clj-http](https://github.com/dakrone/clj-http), a few highlights of http-kit:

1. concurrent requests made easy by asynchronous and promise
2. fast, keep-alive done right, designed with server side use in mind
3. thread safe

## The New API

The API is learn from clj-http: `request`, `get`, `post`, `put`, `head`, `delete`

{% highlight clojure %}
[me.shenfeng/http-kit "2.0-rc1"]
(:require [me.shenfeng.http.client :as http])

; http/post, http/head, http/put, http/delete
; issue http request asynchronous. return a promise, deref to get the response
(http/get url & [options callback])

;; suppoted options
{:timeout 300 ;; read or connect timeout
 :headers {"x-header" "value"}
 :query-params {:ts (System/currentTimeMillis)}
 :user-agent "user-agent-str"
 :basic-auth ["user" "pass"]
 :form-params {:name "http-kit"
               :lang "Clojure"
               :keyword ["async" "concurrent" "fast"]}}
{% endhighlight %}

## Examples

{% highlight clojure %}
;; get synchronously
(let [{:keys [error status headers body]} @(http/get "http://example.com")]
  ;; handle result or error
  )

;; get concurrently, deal one by one
(let [r1 (http/get "http://example.com?page=1")
      r2 (http/get "http://example.com" {:query-params {:page 2}})]
  ;; other keys: :headers, :request, :error
  (println "First page, status: " (:status @r1) "html: " (:body @r1))
  (println "Second page, status: " (:status @r2) "html: " (:body @r2)))

;; get the real urls concurrently
(let [resps (map http/head twitter-shorttened-urls)]
  (doseq [resp resps]
    (let [{:keys [request headers error]} @resp]
      (if-not error ;; Exception: timeout, network error, etc
        (println (:url request) "=>" (:location headers))
        ;; schedule retry?
        ))))

(let [options {:timeout 300ms
               :headers {"x-header" "value"}
               :query-params {:ts (System/currentTimeMillis)}
               :user-agent "user-agent-str"
               :basic-auth ["user" "pass"]
               :form-params {:name "http-kit"
                             :lang "Clojure"
                             :keyword ["async" "concurrent" "fast"]}}]
  (http/post "http://example.com" options
            ;asynchronous with callback
             (fn [{:keys [status error body headers]}]
               ;; handle result or error
               ;; the callback's return value is delivered to the promise
               )))
{% endhighlight %}

## Open source

More information, feature suggestion, bug report on [https://github.com/shenfeng/http-kit](https://github.com/shenfeng/http-kit)

Pull request welcome!
