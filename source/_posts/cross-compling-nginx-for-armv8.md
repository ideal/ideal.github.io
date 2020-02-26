---
title: cross compling Nginx for ARMv8
date: 2020-02-26 15:39:27
tags:
    - cross compiling
---
首先我们需要一个工具链，目前有很多可以选择，此处我们选择[Linaro](https://www.linaro.org/)，这是一种预先构建好的工具链。甚至也可以通过工具构建一个工具链，比如[Buildroot](https://buildroot.org/)，但此处我们不需要那么复杂。

其次按照自己的需求从[这里](https://www.linaro.org/downloads/)来选择一个工具链下载，比如我期望将Nginx运行在[NanoPI Neo2](http://wiki.friendlyarm.com/wiki/index.php/NanoPi_NEO2/zh#.E5.AE.89.E8.A3.85.E7.B3.BB.E7.BB.9F)上面，处理器属于ARMv8，其上运行的系统是基于Ubuntu Core构建（使用glibc），所以我应该选择aarch64-linux-gnu。这个三项代表cpu-vendor-os，在[这里](https://www.gnu.org/software/autoconf/manual/autoconf-2.65/html_node/Specifying-Target-Triplets.html)有说明。

另外要注意具体的工具链版本，比如这个版本 https://releases.linaro.org/components/toolchain/binaries/latest-7/ 里面提到glibc是2.25，但是我那个系统里面的glibc版本并没有这么高，如果用这个来编译，再放到我的NanoPI Neo2里面去运行，会导致运行时GLIBC_2.25符号找不到的错误。所以这里我选择https://releases.linaro.org/components/toolchain/binaries/6.5-2018.12/aarch64-linux-gnu/gcc-linaro-6.5.0-2018.12-x86_64_aarch64-linux-gnu.tar.xz 。

<!-- more -->

假设将其解压到了~/gcc-linaro-6.5.0-2018.12-x86_64_aarch64-linux-gnu ，首先设置一下环境变量：
```shell
$ export PATH=~/gcc-linaro-6.5.0-2018.12-x86_64_aarch64-linux-gnu/bin:$PATH
```

然后下载Nginx 1.15.6，zlib 1.2.11，openssl 1.1.1d，pcre 8.43（当然版本不一定要按照这个来）。Nginx不是使用autotools也不是cmake这样的构建系统，而是自己编写的一套shell脚本，同时它依赖的zlib、openssl、pcre都可以通过源码构建静态链接进去，我这里选择这样做只是为了测试，实际上你也可以依赖于目标系统的动态库。

但由于Nginx的configure并不会把交叉编译信息传递给其依赖的zlib等源码目录构建参数，所以我们这里先进入每个依赖的源码目录，分别把pcre、openssl编译好。

首先编译pcre（注意替换一下/path/to）：
```shell
$ LDFLAGS=-L/path/to/gcc-linaro-6.5.0-2018.12-x86_64_aarch64-linux-gnu/lib CFLAGS=-I/path/to/gcc-linaro-6.5.0-2018.12-x86_64_aarch64-linux-gnu/include CC=aarch64-linux-gnu-gcc CXX=aarch64-linux-gnu-g++ ./configure --disable-shared --build=aarch64-linux-gnu --host=aarch64-linux

$ make
```

之后编译openssl（注意替换一下/path/to）：
```shell
$ ./Configure linux-aarch64 --prefix=/path/to/openssl-1.1.1d/.openssl no-shared no-threads --cross-compile-prefix=aarch64-linux-gnu-

$ make

$ make install
```

最后再来编译Nginx（注意替换一下/path/to）：
```shell
$ ./configure --with-ld-opt=-L/path/to/gcc-linaro-6.5.0-2018.12-x86_64_aarch64-linux-gnu/lib --with-cc-opt=-I/path/to/gcc-linaro-6.5.0-2018.12-x86_64_aarch64-linux-gnu/include --prefix=/usr --with-zlib=/path/to/zlib-1.2.11 --with-pcre=/path/to/pcre-8.43 --with-openssl=/path/to/openssl-1.1.1d --with-http_ssl_module

$ make CC=aarch64-linux-gnu-gcc
```

可以安装到某个目录：
```shell
$ make install DESTDIR=/tmp/nginx
```

通过file可以看到针对的处理器确实是ARM aarch64：
![file nginx](/images/file-nginx.png)

最后可以把整个/tmp/nginx上传到NanoPi Neo2上面进行测试（如果没有网络的话，可以通过screen、minicom、C-Kermit之类的工具，通过usb2uart连接上去。)

![nginx running](/images/nginx-running.png)

