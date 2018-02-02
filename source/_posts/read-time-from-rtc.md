---
title: read time from rtc
date: 2018-02-02 16:39:48
tags:
    - rtc
    - x86
    - linux
---

[这本书](https://read.douban.com/ebook/15160895/)里提到，在内核初始化的过程中，会通过内联汇编从RTC读取当前时间。大概逻辑是：通过汇编，分别读取当前秒、当前分钟等。每次读取，先针对IO端口0x70设置读取的寄存器，然后从0x71读出相应的字节。再进行一定的转换。然后调用mktime()得到一个unix时间戳。
