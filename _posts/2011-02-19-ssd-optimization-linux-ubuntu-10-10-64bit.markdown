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

1.  修改/etc/fstab, 为每个分区加上 noatime,data=writeback。 / 分区需要运行一下:
    > sudo tune2fs -o journal\_data\_writeback /dev/sda1

    其中/dev/sda1 为 / 分区
2.  edit /etc/fstab： tmpfs /tmp tmpfs defaults 0 0 tmpfs /var/log
    tmpfs defaults 0 0 tmpfs /var/tmp tmpfs defaults 0 0 tmpfs
    /home/feng/Downloads tmpfs defaults 0 0
3.  修改/etc/default/grub GRUB\_CMDLINE\_LINUX="elevator=noop"
    GRUB\_TERMINAL=console GRUB\_TIMEOUT=2
4.  when use apt-get: sudo mount tmpfs /var/cache/apt/archives -t
    tmpfs mkdir /var/cache/apt/archives/partial
5.  edit ﻿﻿﻿/etc/X11/Xsession: ERRFILE=/tmp/.xsession-errors
6.  browser cache: firefox: browser.cache.disk.parent\_directory =
    /tmp chrome: --user-data-dir=/tmp



