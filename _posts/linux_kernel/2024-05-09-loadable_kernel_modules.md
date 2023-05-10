---
title: Loadable Kernel Modules
date: 2023-05-08
categories: [linux_kernel]
tags: [setup]
---
# Overview

Whether you are fuzzing or looking into certain Linux subsystems, you might need to set up a loadable kernel module if you selected `m` option on kernel config. An example would be something like `Netfilter` which would load that module when you use them dynamically. If you do an `lsmod` you should be able to see some of loadable kernel modules like `nfnetlink` (Netfilter component).  These aren't automatically there when you follow this [syzkaller guide](https://github.com/google/syzkaller/blob/master/docs/linux/setup_ubuntu-host_qemu-vm_x86-64-kernel.md). In this blog, we will compile kernel and setup Qemu image so that it can load these modules.

## lsmod
![](/assets/img/2023-05-09-23-24-48.png)

## modinfo

![](/assets/img/2023-05-09-23-24-56.png)

# Compiling Kernel and Image

Just follow this [syzkaller guide](https://github.com/google/syzkaller/blob/master/docs/linux/setup_ubuntu-host_qemu-vm_x86-64-kernel.md). By default, Debian image created by `create-image.sh` will be 2G. Sometimes, this is not big enough especially if you compile the kernel with something like Ubuntu config. You can actually expand this image by doing `qemu-img resize stretch.img +20G` (This will increase the image size by 20G but you can change the number).

Then you should also increase the size insize linux guest. with tools like `parted` and `resize2fs`

```sh
parted /dev/sda resizepart 1 100%
resize2fs /dev/sda
```

# Compiling Kernel Modules

You need to compile Linux kernel module by doing
```sh
make modules
```

Then Save it by doing the command below. You can replace `INSTALL_MOD_PATH` with the path you want. 

```sh
INSTALL_MOD_PATH=./linux_modules make modules_install
```

Then move the linux_modules folder to Qemu host using scp.
In guest Linux, move the `/lib` folder under linux_modules to system `/lib`. It should look something like below. Note that the folder path needs to match `uname -r`

![](/assets/img/2023-05-09-23-25-06.png)

Once that is done do and restart. 

```
depmod -a
```

If you do `lsmod`, it should have bunch of loadable modules if your kernel is expecting loadable kernel modules.