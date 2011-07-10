---
author: feng
date: '2011-03-05 13:31:31'
layout: post
status: publish
title: Create bootable flash disk to install Windows 7 from Windows
categories:
- technology
---

**From windows:** 1. prepare a 4G or lager flash disk 2. open cmd,
with amdin privileges.
[![diskpart](/imgs/diskpart-create-win7-bootable.jpg "diskpart windows")](/imgs/diskpart-create-win7-bootable.jpg)
3. copy all content of the dvd to flash disk 4. execute bootsect
/nt60 X: (X is the flash disk) **From linux:**
**[http://superuser.com/questions/62506/can-windows-7-boot-from-an-external-usb-or-firewire-drive](http://superuser.com/questions/62506/can-windows-7-boot-from-an-external-usb-or-firewire-drive)**
**Update:** there is a software
: [http://wintoflash.com/](http://wintoflash.com/) , it can
automate the procedure.


