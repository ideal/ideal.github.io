---
title: read time from rtc
date: 2018-02-02 16:39:48
tags:
    - rtc
    - x86
    - linux
---

[这本书](https://read.douban.com/ebook/15160895/)里提到，在内核初始化的过程中，会通过内联汇编从RTC读取当前时间。大概逻辑是：通过汇编，分别读取当前秒、当前分钟、当前小时等。每次读取，先针对IO端口0x70设置读取的寄存器，然后从0x71读出相应的字节。再进行一定的转换。然后调用kernel_mktime()得到一个unix时间戳。

至于为什么是0x70，[这里](https://wiki.osdev.org/CMOS)有相关的说明。通过`/proc/ioports`也可以看到。

```shell
$ cat /proc/ioports
0000-0cf7 : PCI Bus 0000:00
    0000-001f : dma1
    0020-0021 : pic1
    0040-0043 : timer0
    0050-0053 : timer1
    0060-0060 : keyboard
    0064-0064 : keyboard
    0070-0077 : rtc0
```

RTC对应的IO端口地址正是0x70~0x77。

<!-- more -->

大致代码如下：

```c
#include <stdio.h>
#include <time.h>

#define outb_p(value, port)                \
    __asm__(                               \
        "outb %%al, %%dx\n"                \
        "jmp 1f\n"                         \
        "1: jmp 1f\n"                      \
        "1:" : : "a"(value), "d"(port)     \
    )

#define inb_p(port) ({                     \
    unsigned char _v;                      \
    __asm__ __volatile__(                  \
        "inb %%dx, %%al\n"                 \
        "jmp 1f\n"                         \
        "1: jmp 1f\n"                      \
        "1: ": "=a"(_v) : "d"(port));      \
        _v;                                \
     })

#define CMOS_READ(addr) ({   \
    outb_p(0x80|addr, 0x70); \
    inb_p(0x71);             \
    })

#define BCD_TO_BIN(val) \
    ((val) = ((val) & 15) + ((val)>>4) * 10)

int main(int argc, char *argv[])
{
    struct tm time = { 0 };

    time.tm_sec = CMOS_READ(0);
    time.tm_min = CMOS_READ(2);
    time.tm_hour = CMOS_READ(4);
    time.tm_wday = CMOS_READ(6);
    time.tm_mday = CMOS_READ(7);
    time.tm_mon = CMOS_READ(8);
    time.tm_year = CMOS_READ(9);

    BCD_TO_BIN(time.tm_sec);
    BCD_TO_BIN(time.tm_min);
    BCD_TO_BIN(time.tm_hour);
    BCD_TO_BIN(time.tm_wday);
    BCD_TO_BIN(time.tm_mday);
    BCD_TO_BIN(time.tm_mon);
    BCD_TO_BIN(time.tm_year);

    time.tm_mon--;

    printf("%s\n", asctime(&time));

    return 0;
}
```

0, 2, 4这些表示要读取的地址，会被写入0x70。至于为什么要或上0x80，有这么一句话：Whenever you send a byte to IO port 0x70, the high order bit tells the hardware whether to disable NMIs from reaching the CPU. If the bit is on, NMI is disabled (until the next time you send a byte to Port 0x70). The low order 7 bits of any byte sent to Port 0x70 are used to address CMOS registers.

然后从0x71去读取一个字节。而且需要进行转换显示。读出的数是一种BCD模式。比如49秒，读出的数字是0x49 == 73，需要反向的转换一下。其实是把读出的数字转换成16进制。

编译执行发现出现段错误，调试发现是`outb`那里就出错了。猜测是无法直接对这些地址进行操作。后来发现`sys/io.h`里面包含了outb()以及outb_p()等等的定义。查了outb的man手册，有这么一句话：You use ioperm(2) or alternatively iopl(2) to tell the kernel to allow the user space application to access the I/O ports in question.

改为如下方式：

```c
#include <stdio.h>
#include <time.h>
#include <sys/io.h>

#define CMOS_READ(addr) ({   \
    outb_p(0x80|addr, 0x70); \
    inb_p(0x71);             \
    })

#define BCD_TO_BIN(val) \
    ((val) = ((val) & 15) + ((val)>>4) * 10)

int main(int argc, char *argv[])
{
    struct tm time = { 0 };

    iopl(3);

    time.tm_sec = CMOS_READ(0);
    time.tm_min = CMOS_READ(2);
    time.tm_hour = CMOS_READ(4);
    time.tm_wday = CMOS_READ(6);
    time.tm_mday = CMOS_READ(7);
    time.tm_mon = CMOS_READ(8);
    time.tm_year = CMOS_READ(9);

    BCD_TO_BIN(time.tm_sec);
    BCD_TO_BIN(time.tm_min);
    BCD_TO_BIN(time.tm_hour);
    BCD_TO_BIN(time.tm_wday);
    BCD_TO_BIN(time.tm_mday);
    BCD_TO_BIN(time.tm_mon);
    BCD_TO_BIN(time.tm_year);

    time.tm_mon--;

    printf("%s\n", asctime(&time));

    return 0;
}
```

iopl()是这是IO的权限级别，默认是0，无法访问那些IO端口。编译之后，发现还是段错误，原因是没有root权限，依然无法操作。

```shell
$ sudo ./time
Mon Feb  5 14:17:37 1918
```

sudo方式可以执行成功，但是年份不对。原因是后来加入了年份的世纪信息。具体可以查看[这里](https://wiki.osdev.org/CMOS)。

在这里，iopl(3)也可以改成
```c
ioperm(0x70, 2, 1);
```

但是发现执行失败，看了下io.h里面outb_p()的定义，发现还写入了0x80这个地址，所以可以再加一行：

```c
ioperm(0x80, 1, 1);
```

或者改为调用outb()。另外文档上说ioperm()的第二个参数表示bits，我试了却发现应该是bytes。

