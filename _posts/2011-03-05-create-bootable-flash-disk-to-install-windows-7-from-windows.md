---
author: feng
date: '2011-03-05 13:31:31'
layout: post
status: publish
title: Create bootable flash disk to install Windows 7 from Windows
categories:
- technology
---

**From windows:** 
1. prepare a 4G or lager flash disk 
2. open cmd, with amdin privileges.

[![diskpart](/imgs/diskpart-create-win7-bootable.jpg "diskpart
windows")](/imgs/diskpart-create-win7-bootable.jpg)

3. copy all content of the dvd to flash disk 
4. execute 
{% highlight sh %}
  bootsect /nt60 X:  #(X is the flash disk) 
{% endhighlight %}
