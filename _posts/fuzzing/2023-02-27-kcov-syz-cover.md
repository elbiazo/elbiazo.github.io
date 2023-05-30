---
title: Visualizing KCOV with syz-cover
date: 2023-04-27
categories: [fuzzing]
tags: [syzkaller, kcov, syz-cover]
---

## Overview

If you have used syzkaller, you seen how they have visualizer for kernel [coverage](https://github.com/google/syzkaller/blob/master/docs/coverage.md). You can actually use [syz-cover](https://github.com/google/syzkaller/blob/master/tools/syz-cover/syz-cover.go) to do this with any [kcov](https://docs.kernel.org/dev-tools/kcov.html).

### Syzkaller Coverage Viewer

![](/assets/img/2023-04-27-22-33-53.png)

## Building syz-cover

### Prereq
If you never installed syzkaller before follow this [guide](https://github.com/google/syzkaller/blob/master/docs/linux/setup.md#go-and-syzkaller)

### Build syz-cover (cover)
```sh
git clone https://github.com/google/syzkaller
cd syzkaller
make cover 
```

syz-cover should be under `syzkaller/bin`.

### Building Kernel and Running it on QEMU

Follow syzkaller (guide)[https://github.com/google/syzkaller/blob/master/docs/linux/setup_ubuntu-host_qemu-vm_x86-64-kernel.md] to setup the environment. Most important part is making sure to enable `CONFIG_KCOV=y` in kernel config.

### Building Test Program

I will be using modified version of code shown in official kcov [guide](https://docs.kernel.org/dev-tools/kcov.html)
Code it will try to get kernel coverage for is `read(-1, NULL, 0);`

Modification is we subtract `5` from kcov address, which effectively gives us the previous instruction. kcov by default gives you addres after `CALL __sanitizer_cov_trace_pc` but syz-cover parses address at `CALL` instruction. 5 bytes come from `CALL sanitizer_cov_trace_pc` being 5 bytes. 

Code that calculates the PreviousInstructionPC is [here](https://github.com/google/syzkaller/blob/70a605de85f9d197b61ec86d50dd98b91a39e585/pkg/cover/backend/pc.go#L16)

Copy this code below and compile it by doing `g++ -static -o kcov_test kcov_test.cc`

### kcov_test.cc

```cpp
#include <stdio.h>
#include <stddef.h>
#include <stdint.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <unistd.h>
#include <fcntl.h>
#include <linux/types.h>
#include <iostream>
#include <fstream>

#define KCOV_INIT_TRACE _IOR('c', 1, unsigned long)
#define KCOV_ENABLE _IO('c', 100)
#define KCOV_DISABLE _IO('c', 101)
#define COVER_SIZE (64 << 10)

#define KCOV_TRACE_PC 0
#define KCOV_TRACE_CMP 1

#define AMD64_PREV_INSTR_SIZE 5

int main(int argc, char **argv)
{
        int fd;
        unsigned long *cover, n, i;

        /* A single fd descriptor allows coverage collection on a single
         * thread.
         */
        fd = open("/sys/kernel/debug/kcov", O_RDWR);
        if (fd == -1)
                perror("open"), exit(1);
        /* Setup trace mode and trace size. */
        if (ioctl(fd, KCOV_INIT_TRACE, COVER_SIZE))
                perror("ioctl"), exit(1);
        /* Mmap buffer shared between kernel- and user-space. */
        cover = (unsigned long *)mmap(NULL, COVER_SIZE * sizeof(unsigned long),
                                      PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
        if ((void *)cover == MAP_FAILED)
                perror("mmap"), exit(1);
        /* Enable coverage collection on the current thread. */
        if (ioctl(fd, KCOV_ENABLE, KCOV_TRACE_PC))
                perror("ioctl"), exit(1);
        /* Reset coverage from the tail of the ioctl() call. */
        __atomic_store_n(&cover[0], 0, __ATOMIC_RELAXED);

        /* That's the target syscal call. */
        read(-1, NULL, 0);

        /* Read number of PCs collected. */
        n = __atomic_load_n(&cover[0], __ATOMIC_RELAXED);

        std::ofstream kcov_s("./kcov.txt");

        for (i = 0; i < n; i++)
        {
                /* Make sure address leads with 0x and substract 5 to get previous instruction address for syz-cover parser */
                kcov_s << std::showbase << std::hex << cover[i + 1] - AMD64_PREV_INSTR_SIZE << std::endl;
        }
        
        /* Disable coverage collection for the current thread. After this call
         * coverage can be enabled for a different thread.
         */
        if (ioctl(fd, KCOV_DISABLE, 0))
                perror("ioctl"), exit(1);
        /* Free resources. */
        if (munmap(cover, COVER_SIZE * sizeof(unsigned long)))
                perror("munmap"), exit(1);
        if (close(fd))
                perror("close"), exit(1);
        return 0;
}
```


## Running Test Binary

Transfer kcov_test to vm and run it.

From same [syzkaller installtion guide](ssh -i $IMAGE/bullseye.id_rsa -p 10021 -o "StrictHostKeyChecking no" root@localhost)

it tells you how you can ssh and scp to the vm

### SSH
```sh
ssh -i $IMAGE/bullseye.id_rsa -p 10021 -o "StrictHostKeyChecking no" root@localhost
```

### SCP

```sh
scp -i $IMAGE/stretch.id_rsa -P 10021 -o "StrictHostKeyChecking no" <transfer program> root@localhost:<transfer_path>
```

## Running syz-cover on kcov.txt

kcov_test should have created kcov.txt under same directory. Grab kcov.txt from VM to your host so you can run syz-cover.

syz-cover should look something like this below. It needs to start with `0x`. Also, address should all point to `CALL __sanitizer_cov_trace_pc`. Like mentioned earlier, kcov by default gives you addres after `CALL __sanitizer_cov_trace_pc` but syz-cover parses address at `CALL` instruction.

```
0xffffffff81b3a121
0xffffffff81b39f08
0xffffffff81bb41f1
0xffffffff81bb0064
0xffffffff81bb02e4
0xffffffff81bb02c1
0xffffffff81bb4244
0xffffffff81b3a015
0xffffffff810e53a7
0xffffffff810e53d8
0xffffffff810e5418
0xffffffff810e5400
```

### Disassembly at 0xffffffff81b3a121 (First KCOV)

![](/assets/img/2023-04-27-22-34-06.png)

### Running syz-cover

Note that the reason why you need absolute path is because there is bug where it will break if you don't in earlier version of this tool. Fixed in this [issue](https://github.com/google/syzkaller/issues/3686)
```sh
syz-cover --kernel_src <absolute_linux_kernel_path> --kernel_obj <absolute_linux_kernel_path> kcov.txt                                      
```

### Check Result

Make sure `SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)` is highlighted under `fs/read_write.c`

![](/assets/img/2023-04-27-22-34-14.png)