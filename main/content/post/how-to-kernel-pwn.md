+++
author = "Emily Miller"
title = "How to kernel pwn challenges"
date = "2025-10-03"
description = "A crash course in CTF infrastructure for Kernel pwn"
categories = [ "misc" ]
+++


## Overview

This guide will 
- Walk through using the [Buildroot](https://buildroot.org/) tool to build a base system image
- Adding custom components to the stock image (namely custom kernel modules and init scripts)
- Discuss enabling / disabling specific kernel protections
- Demonstrate the use of containerization to deploy as a CTF challenge
- Show a prototype of a fully reproducible kernel challenge build system

## Getting Started

### Building your first image

To start, go to the the [buildroot download's page](https://buildroot.org/download.html) to download an LTS version of buildroot. From here, you can unzip it with `tar -xvf /path/to/my/file.tar.gz`.

Next, navigate to the local directory with your copy of buildroot. 

Buildroot is a tool that allows you to easily build images for a variety of linux compatable devices. It was originally designed to make embedded linux easy to get started with, but also serves as a great tool for kernel pwn challenge creation. 

The first step to use the tool is to build out a config file. When getting started, you can use one of the default config files provided. To see a list, you can run `make list-defconfigs`. In our case, we'll run the following to get the default config for `x86-64`:
```bash
make qemu_x86_64_defconfig
```
Alternatively, you can run the `make menuconfig` to get a menu for configuration, or `make allnoconfig` to use a completely custom config file.

From here, we can build a copy of linux with:
```bash
make -j<max logical core number>`
```
The output will be in `output/images/`.

### Running your image

This output image will, by default, contain a `bzImage` file, which holds the kernel itself, an `rootfs.ext4` file, which holds the root filesystem, and a `start-qemu.sh` script which will boot qemu for you.

Now, when you launch you'll see a login screen. From here, I'd recommend setting up a custom user for the challenge. In this case, you can change the shell for the `root` user to `/bin/false`, and add a new user to `/etc/passwd` and `/etc/shadow`.

### Debugging your kernel

Now that you can run your image, how do you actually attach a debugger to it? In qemu, there exists a `-gdb $PORT` flag which exposes a gdbstub on the provided port (notably, `-s` is short for `-gdb 1234`). You can also pass the flag `-S` which will start the VM in a stopped state, so it will wait for a `continue` instruction from `gdb` before continuing. In order to attach to this you can launch a `gdb` instance, and use the commands `target remote localhost:1234` followed by `continue`. In this case, this means editing your `start-qemu.sh` script to include the `-s -S` flags.

To properly have symbols in the kernel, you'll want to extract the `vmlinux` from your kernel. You can extract this from your `bzImage` file using the [extract-vmlinux script](https://github.com/torvalds/linux/blob/master/scripts/extract-vmlinux). Note that you will need to have compiled your kernel with symbols, as can be set in buildroot through `Kernel hacking --> Compile-time checks and compiler options --> [*] Compile the kernel with debug info`.

In pwntools, once you've edited your `start-qemu.sh` script and extracted the `vmlinux`, you can then automate this like so,

```python
from pwn import *

context.arch = "amd64"

gdbscript = """
target remote localhost:1234
continue
"""

io = process("./start-qemu.sh")

gdb.attach(io, gdbscript=gdbscript, exe="./vmlinux")

io.interactive()
```

Inside the VM, it may also be desirable to be able to see pointer offsets and such at runtime (i.e. reading from `/proc/kallsyms`, checking `dmesg` if this isn't already allowed, etc.). For this, you'll need root in the VM, which is usually done by patching the init script in the challenge to not drop privileges or create a root user account for yourself. See below for writing custom init scripts, though oftentimes if you're simply working on a challenge, the challenge author will have done most of this for you. You just need to build yourself some way of elevating privileges to root.

## Building a challenge

### Writing your kernel module

Writing kernel modules is often _hard_ for beginners, because it's fundamentally working within a much larger codebase that's sparsely documented and can have disasterous consequences. For simpler CTF challenges, you really don't need much more than the basics. As a starting point, I've included the example below, which allows you to write directly into a buffer with no size check.

```C
#include <linux/kernel.h>
#include <linux/uidgid.h>
#include <linux/module.h>
#include <linux/proc_fs.h>
#include <linux/uaccess.h>
#include <linux/cred.h>
#include <linux/sched.h>
#include <linux/init_task.h>


MODULE_DESCRIPTION("My suuuuper secure kernel module");
MODULE_AUTHOR("emily747");
MODULE_LICENSE("GPL");

#define PROC_NAME "challenge"

static ssize_t challenge_write(struct file *file,
                          const char __user *user_buffer,
                          size_t count,
                          loff_t *ppos)
{
    char stack_buffer[64];
    char* ptr = stack_buffer;

    printk(KERN_INFO "[challenge] Writing %zu bytes\n", count);
    if (!access_ok(user_buffer, count))
        return -EFAULT;
    __copy_from_user(ptr, user_buffer, count);

    return count;
}

static const struct proc_ops proc_fops = {
    .proc_write = challenge_write,
};

static struct proc_dir_entry *proc_entry;

int challenge_init(void)
{
    proc_entry = proc_mkdir("challenge", NULL);
    proc_create("kboff", 0666, proc_entry, &proc_fops);

    printk(KERN_INFO "[challenge] Hello!\n");
    printk(KERN_INFO "[challenge] Try to connect to me at /proc/challenge/kboff.\n");
    return 0;
}

void challenge_exit(void)
{
    printk(KERN_INFO "[challenge] Goodbye!\n");
}

module_init(challenge_init);
module_exit(challenge_exit);
```

### Compiling your kernel module

You'll need to use use the kernels build system to build your challenge against the kernel's specific headers. An example `Makefile` can be seen below:

```Makefile
obj-m += challenge.o

# Disables protections for easy challenge
ccflags-y += -fno-stack-protector
ccflags-y += -U_FORTIFY_SOURCE
ccflags-y += -fno-pie
ccflags-y += -no-pie

all:
	$(MAKE) -C $(KERNELDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KERNELDIR) M=$(PWD) clean
```

This can be run like `make KERNELDIR=/path/to/kernel/headers/`, and will output a `challenge.ko` file.

Importantly, this is both architecture and version dependent. In effect, you'll either want to compile on the target itself (feasible if it's a larger VM for researching) or cross-compile with headers from that VM. In the below examples, I mostly do the latter, predominantly because it's significantly faster and avoids needing to run our compilation stage in the VM itself.

### Adding custom components on startup

To add custom components on startup, use a custom `init` script. In order to pass a custom `init` script, we need put the script in the filesystem itself. You can either build this with the kernel or patch the filesystem after the fact. The latter is the approach I'd recommend taking for CTFs, because you'll often want an easy and quick way of updating `init`, modules, etc., and waiting for a full recompile takes a lot of time.

### Adding your custom kernel module

I'd recommend placing most of your custom code in the same place. So, you'll want to copy your compiled `chall.ko` into something like `/challenge/chall.ko`. Anecdotally people tend to dislike it when they have to tear apart a filesystem looking for the module. In my opinion it's quite a useful skill (and quite easy, a simple `grep` for the file extension would do, but I tend to include the specific `chall.ko` as a file in the distributable seperately).

### Writing an init script

Assuming you've kept the default settings, and you have a `rootfs.ext4` file, you can simply edit the file at `/sbin/init` to be your desired script. A template like the following is fairly standard to work with, and note that the script must switch to a long running process or the kernel will kill itself.
```bash
#!/bin/sh

# Insert custom kernel module
insmod /challenge/chall.ko

# Drop into busybox init
exec busybox init
```

### Adding a custom userspace binary

Notice in the above script that the actual script switches to a login shell at the final step in the `busybox init`. If you wanted, you could simply rebuild the init script from scratch to launch directly into your binary. This would allow you to give a userspace pwn into kernel pwn if you wanted to (for instance, shellcoding into kernel pwn). Alternatively, you can set a users shell to your custom binary, which would launch them directly into the userspace program.

If you wanted to build a jail breakout challenge, this same process could be used to provide kernel isolation. A good example of this would be a seccomp restricted shellcoding challenge, where you need to disable seccomp by exploiting a kernel module. Some examples of this format are available on [pwn.college](https://pwn.college/system-security/kernel-security/).

### How do I disable \[X\] protection?

Some protections can be enabled or disabled at runtime (assuming a build with support for it). These include KASLR, SMAP, SMEP, NX, and PTI (though, this can change depending on the architecture --- this is generally the case for x86/x86-64 though). These can be disabled by adding flags in qemu. For instance, to disable KASLR, you can use `-append "nokaslr"` (multiple options are delimited with a space), which will disable KASLR at runtime.

Some protections can only be changed at compile time, namely canaries and SLUB/SLAB hardening. These can be changed by editing the `CONFIG_STACKPROTECTOR`, `CONFIG_SLAB_FREELIST_HARDENED`, etc. options in your `kconfig`. The easiest way to use a custom `kconfig` with buildroot is to use the menu build option (`make menuconfig`). 

## Running with docker!

Next, we can create a Dockerfile to actually host the challenge, which can be done fairly simply with one like so,
```Dockerfile
FROM ubuntu:latest AS bootstrap
WORKDIR /app/
RUN apt-get update && apt-get install -y qemu-system
COPY ./start-qemu.sh ./run
COPY rootfs.ext2 ./rootfs.ext2
COPY bzImage ./bzImage

FROM pwn.red/jail:0.4.1@sha256:ee52ad5fd6cfed7fd8ea30b09792a6656045dd015f9bef4edbbfa2c6e672c28c
COPY --from=bootstrap / /srv
ENV JAIL_TIME=600
ENV JAIL_MEM=384M
```
At this point, depending on your build configurations, it may be necessary to tweak `start-qemu.sh` (for instance, to adjust filesystem mounting options or enabling / disabling runtime kernel protections).

## Running with docker (but unhinged)!

To take this a step further, we can completely automate the build process, so that the entire thing is done in Docker. The major benefit of this is completely reproducable builds, but the major downside being the fact that it relies on docker caching layers to stop the kernel from fully rebubilding itself every time. In production, I recommend splitting this into two (one for building the filesystem / kernel, and the other for running VM). An example of this can be seen below:

```Dockerfile
FROM debian:unstable@sha256:cc1675ddb1073d19ba9ef6fe9b9c625eceb02fccb9c0f7afbb4e60f16325c91d AS fs_builder
RUN apt-get update && apt-get install -y which sed make binutils \
    build-essential diffutils gcc g++ bash patch gzip bzip2 perl \
    tar cpio unzip rsync file bc findutils gawk wget git
RUN git clone https://github.com/buildroot/buildroot.git /tmp/buildroot
COPY .config /tmp/buildroot/.config
RUN --mount=type=cache,target=/tmp/buildroot/dl/,id=buildroot_dl \
    cd /tmp/buildroot && make -j8 && \
    mkdir -p /tmp/artifacts && \
    cp -r /tmp/buildroot/output/build/linux-* /tmp/artifacts/ && \
    cp /tmp/buildroot/output/images/rootfs.ext2 /tmp/artifacts/ && \
    cp /tmp/buildroot/output/images/bzImage /tmp/artifacts/ && \
    cp /tmp/buildroot/output/images/start-qemu.sh /tmp/artifacts/

# Build the kernel module against the exact kernel buildroot produced
FROM debian:unstable@sha256:cc1675ddb1073d19ba9ef6fe9b9c625eceb02fccb9c0f7afbb4e60f16325c91d AS mod_builder
RUN apt-get update && apt-get install -y build-essential libelf-dev
COPY --from=fs_builder /tmp/artifacts/linux-* /tmp/linux/
COPY module/* /tmp/module/
RUN cd /tmp/module && make KERNELDIR=/tmp/linux

FROM debian:unstable@sha256:cc1675ddb1073d19ba9ef6fe9b9c625eceb02fccb9c0f7afbb4e60f16325c91d AS fs_patcher
RUN apt-get update && apt-get install -y e2tools
COPY --from=fs_builder /tmp/artifacts/rootfs.ext2 /tmp/rootfs.ext2
COPY init /tmp/init
COPY --from=mod_builder /tmp/module/*.ko /tmp/module.ko
RUN e2rm /tmp/rootfs.ext2:/sbin/init
RUN e2cp -p /tmp/init /tmp/rootfs.ext2:/sbin/
RUN e2cp -p /tmp/module.ko /tmp/rootfs.ext2:/root/
RUN e2cp /tmp/rootfs.ext2:/etc/passwd /tmp/passwd
RUN echo 'ctf:x:1000:1000::/home/ctf:/bin/sh' >> /tmp/passwd
RUN e2cp -p /tmp/passwd /tmp/rootfs.ext2:/etc/passwd
RUN e2cp -p /tmp/rootfs.ext2:/etc/group /tmp/group
RUN echo 'ctf:x:1000:' >> /tmp/group
RUN e2cp -p /tmp/group /tmp/rootfs.ext2:/etc/group
RUN e2mkdir /tmp/rootfs.ext2:/home/ctf

FROM debian:unstable@sha256:cc1675ddb1073d19ba9ef6fe9b9c625eceb02fccb9c0f7afbb4e60f16325c91d
RUN apt-get update && apt-get install -y --no-install-recommends qemu-system-x86 socat e2tools && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=fs_builder /tmp/artifacts/bzImage ./
COPY --from=fs_builder /tmp/artifacts/start-qemu.sh ./
COPY --from=fs_patcher /tmp/rootfs.ext2 ./
RUN sed -i "s/console=tty1/rw nokaslr/" start-qemu.sh
RUN sed -i "s/-drive file=rootfs.ext2,if=virtio,format=raw/-drive file=rootfs.ext2,if=virtio,format=raw -snapshot/" start-qemu.sh
COPY entrypoint.sh ./
ENV FLAG=PWNED{FAKE_FLAG}
RUN echo "PWNED{FAKE_FLAG}" > /tmp/flag.txt
EXPOSE 5000
CMD ["/app/entrypoint.sh"]
```

You'll also need to populate `entrypoint.sh`, `init`, `.config`, `module/challenge.c`, and `module/Makefile`. In this case, my entrypoint looks something like the following:

```sh
#!/bin/sh
# Inject flag (mounted by scenario) into rootfs
e2cp /tmp/flag.txt /app/rootfs.ext2:/root/flag.txt
rm /tmp/flag.txt

# Each connection gets its own QEMU with -snapshot (reads base image, writes to tmpfile)
exec socat TCP-LISTEN:5000,reuseaddr,fork EXEC:"/app/start-qemu.sh --serial-only",stderr
```

## More guides and resources

- Much of my learning was with [smallkirby's guide](https://github.com/smallkirby/kernelpwn).
- While it can be a lot, [QEMU's official docs are fairly complete for specific questions](https://www.qemu.org/docs/master/)

