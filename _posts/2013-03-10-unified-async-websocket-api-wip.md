---
author: feng
date: '2013-03-10 0-0-00'
layout: post
status: publish
title: Unified API for WebSocket, HTTP streaming and long-polling (WIP)
categories: ['http-kit', 'clojure']
---

I and [Peter](https://github.com/ptaoussanis) were working on a new unified API for WebSocket, HTTP streaming and long-polling with protocol `Channel` and macro `with-channel`.

The new API is fully working now. I just get the time to write a page to explain it a little bit: [http://http-kit.org/channel.html](http://http-kit.org/channel.html).

It has a few benefits:

* unified, simpler
* more feature: support HTTP streaming
* Sever also get notified when client close/away for long-polling/streaming

If interested, please have a try, and tell me how do you think.
