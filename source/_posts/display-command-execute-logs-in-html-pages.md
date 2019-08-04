---
title: display command execute logs in html pages
date: 2019-08-04 21:45:29
tags:
    - node
---

有些时候需要把命令行程序的日志在页面中展示，比如[这种场景](https://travis-ci.org/python/mypy/jobs/565354133)。通常日志里可能含有进度的展示，比如curl的输出，这种展示依赖于\r这种[控制字符](https://en.wikipedia.org/wiki/Control_character)来起作用。遗憾的是如果直接作为html来输出，\r会被当作换行，导致进度变成了无数行，比如[这里](https://travis-ci.org/python/mypy/jobs/565354133#L514)的样子（虽然可能也不是大问题）。另外终端里也会有很多依赖[转义序列](https://en.wikipedia.org/wiki/ANSI_escape_code)来实现的带色彩的输出，比如`git log`的输出。题外话，有时候stdout的类型，会影响这个特性，比如`git log | less`当加上了管道之后，彩色输出会消失。[这里](https://wiki.archlinux.org/index.php/Color_output_in_console#Reading_from_stdin)有一些绕过的方法。

<!-- more -->

回到正题，我们希望带有\r的输出，只保留最后的部分，像下面这样的效果：

![\r output](/images/progress-output.png)

同时将转义序列的彩色输出，也转换成对应的html。

尝试了好久，最终通过[clearr](https://www.npmjs.com/package/clearr)和[ansi-to-html](https://www.npmjs.com/package/ansi-to-html)这两个包可以比较好的做到，示例代码：

![log display example](/images/log-display-example.png)

