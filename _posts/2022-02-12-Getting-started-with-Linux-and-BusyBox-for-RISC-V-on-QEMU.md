---
title:  "Getting started with Linux and BusyBox for RISC-V on QEMU"
layout: post
categories: engineering
long: true
---

In this blog post, we will discuss emulating 64-bit RISC-V system on QEMU and running Linux and BusyBox on this system. We'll explore step-by-step how to build Linux kernel, QEMU and BusyBox for 64-bit RISC-V target. There is a [guide from RISC-V foundation](https://risc-v-getting-started-guide.readthedocs.io/en/latest/linux-qemu.html) on that topic, but it doesn't cover many important points.



## Sources

Here and below we will assume that at some point we are going to explore, reconfigure, debug or even modify the Linux/BusyBox/QEMU source code, so we will build everything from source:
```sh
git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
git clone git@gitlab.com:qemu-project/qemu.git
git clone git://git.busybox.net/busybox
```

## Tools

Here and below we will assume that we are on Ubuntu 22.04. In any way, in today's world, we can make almost any popular environment in three lines:

```sh
docker pull ubuntu:22.04
docker run -ti --rm -v $(pwd):/data ubuntu /bin/bash
```

### Host tools

For being able to build host programs we need a tools and some libraries for the host:
```sh
apt-get update
apt-get install -y build-essential python3 ninja-build pkg-config libglib2.0-dev libpixman-1-dev libslirp-dev flex bison bc file device-tree-compiler
```

### Target tools

Since the target architecture (RISC-V 64-bit) is different from the host architecture, we need a toolchain for cross-compilation:
```sh
apt-get install -y gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
```

## Linux

Following commands builds Linux kernel for RISC-V with default configuration:
```sh
cd linux
mkdir build
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- O=build defconfig
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- O=build -j$(nproc)
cd -
```
Now we have `linux/build/arch/riscv/boot/Image` file with Linux kernel for RISC-V 64-bit:
```
$ file -b linux/build/arch/riscv/boot/Image
MS-DOS executable PE32+ executable (EFI application) RISC-V 64-bit (stripped to external PDB)
```

## QEMU

Following commands builds QEMU whic can emulate RISC-V 64-bit target:
```sh
cd qemu
./configure --enable-debug --target-list=riscv64-softmmu --enable-slirp
make -j`nproc`
cd -
```
Now we have `qemu/build/qemu-system-riscv64` which can even directly boot our `Image`:
```sh
./qemu/build/qemu-system-riscv64 -nographic -machine virt -kernel linux/build/arch/riscv/boot/Image
```
At first, [OpenSBI](https://github.com/riscv-software-src/opensbi) is started:

![image](https://user-images.githubusercontent.com/8286747/218286161-bcc3e081-3c3a-47b2-9b6e-7966f282b62b.png)
Then it starts Linux kernel:

![image](https://user-images.githubusercontent.com/8286747/218286116-3e61e719-cb49-4601-bfb5-4d15d9fa1cd5.png)
Unfortunately it ends with kernel panic because we didn't provide a block device for a filesystem mount:

![image](https://user-images.githubusercontent.com/8286747/218286074-8c2043f4-eb55-4c29-9c5b-feca27700d13.png)
 **Press Ctrl-A then X to exit from QEMU.** What we saw above means that we need to create a proper disk image. It is best if it has `init` (although even `/bin/sh` is OK) and other basic programs.

## BusyBox

Following commands builds BusyBox for RISC-V with default configuration:
```sh
CROSS_COMPILE=riscv64-linux-gnu- make -C busybox defconfig
CROSS_COMPILE=riscv64-linux-gnu- make -C busybox -j $(nproc)
```
But it has to be put somewhere, so let's create an empty 256M ext4 image:
```sh
dd if=/dev/zero of=rootfs.img bs=1M count=256
mkfs.ext4 rootfs.img
```
We can mount it and install BusyBox into it:
```sh
mkdir -p rootfs
mount rootfs.img rootfs
CROSS_COMPILE=riscv64-linux-gnu- LDFLAGS=--static make -C busybox install CONFIG_PREFIX=../rootfs
```
We use `LDFLAGS=--static` to obtain static binary. In addition to the BusyBox itself, we should also create following files and directories inside `rootfs`:
```sh
cd rootfs
mkdir -p proc sys dev etc/init.d
touch etc/fstab
echo '#!/bin/sh' > etc/init.d/rcS
chmod +x etc/init.d/rcS
cd -
```
Generally speaking, `etc/init.d/rcS` is a script for initializing the system during the boot process, setting up the environment and executing other startup scripts, but we can leave it empty. Now we can unmount `rootfs`:
```
umount rootfs
```
Now, we have a disk image with ext4 and BusyBox:
```
$ file -b rootfs.img 
Linux rev 1.0 ext4 filesystem data, UUID=83897434-65d1-48c3-9742-6d4fc0376542 (extents) (64bit) (large files) (huge files)
```

## Running BusyBox in QEMU

I would recommend to use the following script for futher experiments:
```sh
#!/usr/bin/bash -x

KERNEL=linux/build/arch/riscv/boot/Image
DRIVE=rootfs.img

./qemu/build/qemu-system-riscv64 \
    -nographic \
    -machine virt \
    -kernel $KERNEL \
    -append "root=/dev/vda rw console=ttyS0" \
    -drive file=$DRIVE,format=raw,id=hd0 \
    -device virtio-blk-device,drive=hd0 \
    $*
```
The VirtIO block device (`-device virtio-blk-device`) as `/dev/vda` gives the guest access to `rootfs.img` on the host. This time `/` is mounted, `init` is started and we have access to the filesystem and basic programs like `cat` and `ls`:

![image](https://user-images.githubusercontent.com/8286747/218285917-8eeffce2-5ec9-411a-8f0c-2c8c5b9c1558.png)

## Bonus part: inspecting device-tree

DTS (Device Tree Source) is a text-based format used to describe the hardware architecture of a device. DTB (Device Tree Binary) is a binary representation of the same information, which is used by the Linux kernel to configure the hardware during boot time. The DTS file is compiled into a DTB file, which is then used by the kernel to configure the hardware.

This is how the device tree can be dumped from the system we just run and decompiled:
```sh
./qemu/build/qemu-system-riscv64 -machine virt -machine dumpdtb=qemu-riscv64.dtb
dtc -I dtb -O dts -o qemu-riscv64.dts qemu-riscv64.dtb
```

For example, there are 8 entries like the following which helps Linux kernel find VirtIO devices such as `-device virtio-blk-device`:
```python
virtio_mmio@10001000 {
	interrupts = <0x01>;
	interrupt-parent = <0x03>;
	reg = <0x00 0x10001000 0x00 0x1000>;
	compatible = "virtio,mmio";
};
```

## Conclusion

We have emulated a 64-bit RISC-V system on QEMU and ran Linux and BusyBox on it. We have explored how to setup tools, build the Linux kernel, QEMU and BusyBox for the 64-bit RISC-V target. This is an important step towards getting familiar with the RISC-V architecture.