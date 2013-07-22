---
author: feng
date: '2013-7-06 08-21-00'
layout: post
status: publish
published: false
desc: 管理buffer，减少copy，是高性能服务器的关键
title: 高性能服务器：buffer matters
categories: ['server', 'go']
---

最近工作上，遇到一些比较挑战的问题，要提高[美团](http://meituan.com)推荐和排序服务的稳定性。挑战来自很多方面：

1. 已有程序已经跑了2年了，有一点历史包袱
2. 美团的业务增长很快，流量与日俱增
3. 已有程序不能scale
