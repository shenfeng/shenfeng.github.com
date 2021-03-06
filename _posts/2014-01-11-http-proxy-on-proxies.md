---
author: feng
date: '2014-01-11 08-21-00'
layout: post
status: publish
title:  代理的代理：上网，抓取利器
---

##  介绍

做为一个网名，每天花不少时间浏览网页，由于某些原因，某些网站，在某个国家访问起来有困难。让人沮丧。采用过的方法，有如下几种：

1. 使用代理（以及某些工具配合某些浏览器插件，如proxy switchy），缺点是，不是特别稳定。
2. 使用VPN，速度慢，特别时访问国内的网站
3. 有些公司，会在路由器或者网关上，做手脚。干净的解决这个问题

另外最近打算加入一家创业公司，需要全网抓数据，并分析。数据也是对方的宝贵资源，不会轻易让你抓的，会通过各种方式封杀：比如ip rate limiting和user rate limiting等。要和对方斗志斗勇。

今天周末，早上起床后，想着：我应该要解决这两个问题，要干净利落，透明。

## 方案

**代理的代理，对代理进行负载均衡**

可以写一个程序，它自己是一个HTTP代理服务器，但相对于一般的代理服务器，它有几个特点：

1.  程序部署在本地（或者内网，供家人或者团队使用）
2.  对于国内的访问（比如baidu，youku，weibo等），它简单的代理请求，overhead小
3.  对于国外的访问（比如某几个社交网站），它把收到的请求，转发给另外的代理服务器
4.  维护一组代理服务器（数千个），对他们进行分组，进行负载均衡，代理切换等，减轻被IP rate limiting的机会，也加快速度
4.  按照HTTP协议，cache一些资源（上网加速）
5.  实现HTTP connect，用于支持HTTPS。由于内网使用，可以不做安全限制（比如仅允许433端口）

这样，它既是一个上网加速器，还能助我透明畅游海外，并且减轻 crawler维护多代理，进行代理切换的工作。

小文件整体cache到redis，大文件metadata存redis，文件存硬盘。哈哈，这样可以从硬盘上收割无数图片和视频，算是额外收获了。

有点意思。

## 可行性调研

上午开始编码，通过实际写code，来验证可行性。HTTP 代理服务器比较简单，HTTP，SOCKS代理协议也比较容易实现。搞了几个小时，程序基本work，可以做到浏览twiter，并且上youku也很快。

## 接下来工作

完善code，让程序更强壮。还有cache到硬盘的图片，好好规划一下，方便收割。 






