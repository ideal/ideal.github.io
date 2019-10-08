---
title: login screen black after upgrading
date: 2019-10-08 21:29:57
tags:
    - archlinux
---

10.7号`pacman -Syu`了一把，重启发现手动编译的内核系统登录界面停在了启动图形系统，然后一直没反应。不过ALT-F2可以登录tty，然后执行`systemctl status sddm`发现sddm并没有启动X。如果手动启动X，会看到两个报错：
```
Internal error: Could not resolve keysym XF86MonBrightnessCycle
Internal error: Could not resolve keysym XF86RotationLockToggle
```
不过这两个报错并不会影响X的功能。X启动之后依然黑屏。

切换到archlinux自带的内核，却又没问题。sddm最近也没有更新过。KDE倒是进行了大更新。由于只剩下内核的差异（手动编译的内核只是增加了CONFIG_DEBUG_INFO=y），于是我将archlinux内核新的config文件复制一份，重新编译了4.18.12，然而重启依然黑屏。

`sudo journal -u -e sddm`可以看到黑屏的情况，缺少了Adding new display on vt1。

![sddm log](/images/archlinux-sddm-log.png)

还未解决。。

