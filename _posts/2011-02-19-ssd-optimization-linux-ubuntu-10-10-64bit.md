---
author: feng
date: '2011-02-19 22:09:12'
layout: post
status: publish
title: Linux SSD 优化 -- for ubuntu 10.10 64bit
categories:
- linux
- technology
tags:
- ssd
- thinkpad
---

The conf I make to make less write to SSD, thus make it run a little
faster. It's quite dangerous, since log are not kept and other
compromises.

On my dev machine, which has 8G memory and a SSD, works quite good,
trade memory and safety for speed.

All my source code are sync to remote git repository. On my dev
machine, which has nothing but code,  I already have safety.


* 修改/etc/fstab, 为每个分区加上 noatime,data=writeback。/dev/sda1为/
  分区
 {% highlight sh %}
 sudo tune2fs -o journal_data_writeback /dev/sda1
 {% endhighlight %}

* edit /etc/fstab：
 {% highlight sh %}
 tmpfs /tmp tmpfs defaults 0 0
 tmpfs /var/log tmpfs defaults 0 0
 tmpfs /var/tmp tmpfs defaults 0 0
 {% endhighlight %}

* 修改/etc/default/grub
 {% highlight sh %}
 GRUB_CMDLINE_LINUX="elevator=noop"
 GRUB_TERMINAL=console #not for speed, it looks better
 GRUB_TIMEOUT=2
 {% endhighlight %}

* before using apt-get:
 {% highlight sh %}
 sudo mount tmpfs /var/cache/apt/archives -t tmpfs
 mkdir /var/cache/apt/archives/partial
 {% endhighlight %}

* browser cache:
  * firefox: browser.cache.disk.parent_directory = /tmp
  * chrome: --user-data-dir=/tmp
