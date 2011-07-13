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

* 修改/etc/fstab, 为每个分区加上 noatime,data=writeback。 / 分区需要运行一下:
 {% highlight sh %}
 sudo tune2fs -o journal_data_writeback /dev/sda1
 {% endhighlight %}
 其中/dev/sda1 为 / 分区
* edit /etc/fstab：
 {% highlight sh %}
 tmpfs /tmp tmpfs defaults 0 0
 tmpfs /var/log tmpfs defaults 0 0
 tmpfs /var/tmp tmpfs defaults 0 0
 {% endhighlight %}
* 修改/etc/default/grub
 {% highlight sh %}
 GRUB_CMDLINE_LINUX="elevator=noop"
 GRUB_TERMINAL=console
 GRUB_TIMEOUT=2
 {% endhighlight %}
* when use apt-get:
 {% highlight sh %}
 sudo mount tmpfs /var/cache/apt/archives -t tmpfs
 mkdir /var/cache/apt/archives/partial
 {% endhighlight %}
* browser cache:
  * firefox: browser.cache.disk.parent_directory = /tmp
  * chrome: --user-data-dir=/tmp



