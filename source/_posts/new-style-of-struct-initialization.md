---
title: new style of struct initialization
date: 2018-10-27 10:32:19
tags:
    - c
---

好吧，其实也不算新，毕竟部分在C89就有了。

```c
struct arg
{
    const char *key;
    const char *val;
};

struct task
{
    int         pid;
    const char *path;
    time_t      start_time;
    struct arg  args[5];
};
```

```c
// C89的初始化，按照定义的成员顺序赋值，未指定的会被初始化为0
struct task t1 = { 20, };

// 也可以不按定义的顺序，指定初始化哪几个成员，同样未指定的会被初始化为0
struct task t2 = { .start_time = 1537844400, .args[1].key = (char *)0xbeef };
```

<!-- more -->

```c
// C99中增加了一种新的赋值方式：

struct task t3;

t3 = (struct task) {
    .start_time = 100,
    .args[1].key = (char *)0xbeef,
    .args[1].val = (char *)0xcafe,
};

```

当然在GCC里面，如果你用`-std=c89`，这些也都能编译通过；加了`-Wpedantic`，才会有相应的警告。
