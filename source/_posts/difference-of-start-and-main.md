---
title: difference of _start and main
date: 2018-09-30 11:30:47
tags:
    - asm
---

如果使用gcc编译，并且连接了C的标准库（crti.o, crt1.o以及libc等），那么此时和编译普通的C程序一样，需要提供一个main的符号。（crt1.o里面有_start，_start会调用__libc_start_main，再调用main）。例如：

```assembly
# example.s

    .text
    .global main
main:
    pushq  %rbp
    movq   %rsp, %rbp

    movq  $file, %rdi
    movq  $59, %rax    # 系统调用号，来自/usr/include/asm/unistd_64.h

    pushq $0
    pushq %rdi
    movq  %rsp, %rsi   # 构造一个execve()需要的argv参数
    movq  $0, %rdx
    syscall

    movl $0, %eax
    leave
    ret

file: .string  "/bin/ps"
```

通过
```shell
$ gcc example.s -o example
```
编译即可。

或者也可以：
```shell
$ as example.s -o example.o && ld -dynamic-linker /lib64/ld-linux-x86-64.so.2 /usr/lib64/crt1.o /usr/lib64/crti.o /usr/lib64/crtn.o -lc example.o -o example
```

当然crt那些文件的路径可能会有些差异。

如果你不需要链接libc，只需要把`main`改成`_start`，然后这样编译：

```shell
$ gcc -nostdlib example.s -o example
```
，或者分成两步：
```shell
$ as example.s  -o example.o && ld example.o -o example
```

此时对生成的文件执行ldd，会提示不是一个动态链接的可执行文件。
