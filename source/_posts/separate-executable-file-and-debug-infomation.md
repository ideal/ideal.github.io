---
title: separate executable file and debug infomation
date: 2018-11-30 01:34:21
tags:
    - debug
---

很多发行版会将包含调试符号信息的文件拆分到另外的安装包里，所以相应的wiki也会教你遇到进程崩溃时，如何获取完整的调用栈信息，其中一步便是安装带有调试信息的软件包，比如[这里](https://wiki.ubuntu.com/DebuggingProgramCrash#Installing_debug_symbols_manually)、[这里](https://wiki.debian.org/HowToGetABacktrace#Installing_the_debugging_symbols)。

至于为什么可以这么做，这其实是GDB的机制。按照[这里](https://sourceware.org/gdb/onlinedocs/gdb/Separate-Debug-Files.html)的描述，GDB支持独立于可执行文件的调试符号信息。

<!-- more -->

从一个二进制文件产生调试信息文件很简单，只需要执行：

```bash
$ objcopy --only-keep-debug execfile execfile.debug

```

然后可以把原文件的调试符号等信息删除：

```bash
$ strip -s execfile

```

但此时如果用gdb调试，它并不能正确地找到调试信息。还需要针对可执行文件添加一个调试信息文件的引用信息：

```bash
$ objcopy --add-gnu-debuglink=execfile.debug execfile

```

通过readelf可以看到原文件多了一个.gnu_debuglink的段：

```bash
$ readelf --headers execfile | grep .gnu_debuglink

```

在此之后，其实可以将execfile.debug放到其他位置去，gdb也能加载到，假设execfile所在目录为/path，可以：

```bash
$ sudo mv execfile.debug /usr/lib/debug/path/

```

该路径通过gdb的debug-file-directory变量控制。这只是其中一个查找路径，实际比这复杂。

另外，通过`readelf -w`可以查看.gnu_debuglink段的信息，它也会自动查找调试文件，但是目录规则和gdb却是不一样的。共同点是都会去查找execfile所在目录的execfile.debug文件。

