---
title: how to calculate page directory and page table size
date: 2018-04-08 12:59:44
tags:
---

Linux 0.11保护模式下能寻址16MB地址空间（虽然地址是32位的）。需要的页表数和页目录表数计算方法：

由于页的大小是4096（2^12)，那么需要的页表项目是（2^24/2^12）= 2^12，也就是4096个页表项。

每个页表项是4字节，那么总共的大小就是2^14个字节，也就是16KB，刚好是4页。

所以可以看到，它填充了4个页的页表。

很明显4个页表，只需要页目录里的4项，就可以定位这些页表了。

也可以这样计算：16KB，也就是2^14，(2^14/2^12) = 4。所以只占用了页目录表的前4项。

![page directory and page table](/images/page-directory-and-page-table.png)

实际上，4KB大小的页目录表，已经能够寻址4GB的空间了。

2^32 / 2^12 = 2^20

每项4字节，也就是需要4MB的页表。

为了寻址4MB的页表：2^22 / 2^12 = 1024，每个4字节，刚好页目录表是4KB。

参考：
1. https://nieyong.github.io/wiki_cpu/CPU%E4%BD%93%E7%B3%BB%E6%9E%B6%E6%9E%84-MMU.html
