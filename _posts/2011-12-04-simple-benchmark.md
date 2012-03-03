---
author: feng
date: '2011-12-04 20-00-00'
layout: post
published: false
title: Simple benchmark
---

I assembly a new desktop today:

CPU: Intel(R) Core(TM) i7-2600
Mem: ADATA DDR3 1333 4G*4
Mainboard: MSI H67MA-E45 （B3）
Hard disk:

dd if=/dev/zero of=/dev/null bs=20M count=1024
1024+0 records in
1024+0 records out
21474836480 bytes (21 GB) copied, 2.50215 s, 8.6 GB/s

dd if=/dev/zero of=/dev/null bs=6M count=1024
1024+0 records in
1024+0 records out
6442450944 bytes (6.4 GB) copied, 0.342609 s, 18.8 GB/s

