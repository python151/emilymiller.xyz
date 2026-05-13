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

## Building your image and running it with qemu

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

This output image will, by default, contain a `bzImage` file, which holds the kernel itself, an `rootfs.ext4` file, which holds the root filesystem, and a `start-qemu.sh` script which will boot qemu for you.

Now, when you launch you'll see a login screen. From here, I'd recommend setting up a custom user for the challenge. In this case, you can change the shell for the `root` user to `/bin/false`, and add a new user to `/etc/passwd` and `/etc/shadow`.

## Adding custom components on startup

This can be done with a custom `init` script. In order to pass a custom `init` script, we need a custom `initramfs`. You can either build this into the kernel, or you can build this and run qemu with flags for a custom initramfs. The latter is the approach I'd recommend taking for CTFs, because you'll often want an easy and quick way of updating init scripts, and waiting for a full recompile takes a lot of time.

### Adding your custom kernel module

I'd recommend placing most of your custom code in the same place. So, you'll want to copy your compiled `chall.ko` into something like `/challenge/chall.ko`.

### Compiling your kernel module

This is a more complex issue, but in short, you can use the Kernel's build system to build your challenge against the Kernel's specific headers. An example `Makefile` can be seen below:

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

Alternatively, if you wanted to build a jail breakout challenge, this same process could be used to provide kernel isolation.

## How do I disable \[X\] protection?

Some protections can be enabled or disabled at runtime (assuming a build with support for it). These include KASLR, SMAP, SMEP, NX, and PTI. These can be disabled by adding flags in qemu. For instance, to disable KASLR, you can use `-append "nokaslr"` (multiple options are delimited with a space), which will disable KASLR at runtime.

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

To take this a step further, we can completely automate the build process, so that the entire thing is done in Docker. An example of this can be seen below:

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

