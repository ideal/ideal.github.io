---
title: simple way to compile kernel on archlinux
date: 2018-10-28 20:53:38
tags:
    - kernel compiling
---

实际上ArchLinux的[Wiki][1]已经写得非常清楚了，这里只是记录、简化和补充。

首先下载一份源代码，比如[这个][2]。同时下载对应的签名，比如[这个][3]。

验证签名：

```bash
$ gpg --list-packets linux-4.19.tar.sign
$ gpg --recv-keys <上一步输出的keyid>
```

第二步之后可以通过`gpg --list-public-keys`查看导入的公钥。

然后进行验证：

```bash
$ unxz linux-4.19.tar.xz
$ gpg --verify linux-4.19.tar.sign linux-4.19.tar
```

<!-- more -->

之后解压linux-4.19.tar然后切换到解压生成的目录。

```bash
$ make clean && make mrproper
```

复制一份config文件：

```bash
$ zcat /proc/config.gz > .config
```

一般来说.config不需要什么修改，假如需要编译一份需要带有调试信息的内核，只需要修改这里：

```ini
CONFIG_DEBUG_INFO=y
```

当然也可以加个局部版本号：

```ini
CONFIG_LOCALVERSION="-fatcat"
```

之后便是编译：

```bash
$ make -j4
```

时间会比较长。编译完成之后：

```bash
$ sudo make modules_install
$ sudo cp -v arch/x86_64/boot/bzImage /boot/vmlinuz-linux-fatcat
```

生成ramdisk：

```bash
$ sudo mkinitcpio -k 4.19.0-fatcat -g /boot/initramfs-linux-fatcat.img
```

之后便是修改增加启动项，如果是grub，只需要：

```bash
$ sudo grub-mkconfig > /boot/grub/grub.cfg
```

另外如果通过dkms安装过virtualbox模块，还需要运行dkms对新的内核编译一份对应的模块：

```bash
$ sudo dkms autoinstall -k 4.19.0-fatcat
```

[1]: https://wiki.archlinux.org/index.php/Kernel/Traditional_compilation
[2]: https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.19.tar.xz
[3]: https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.19.tar.sign
