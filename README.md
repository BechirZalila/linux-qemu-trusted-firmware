# ARMv8 Linux System on QEMU with Trusted Firmware-A (TF-A)

***THIS IS A WORK IN PROGRESS...***

## Introduction

This project provides a step-by-step guide to building an ARMv8-based Linux system on QEMU using:

- **Trusted Firmware-A (TF-A)**: The reference implementation of the secure world firmware for ARMv8.
- **U-Boot Bootloader**: The widely used bootloader for embedded systems.
- **Linux Kernel**: Version 6.1.128, compiled manually for ARMv8.
- **Custom Root Filesystem**: A BusyBox-based root filesystem.
- **QEMU**: An ARMv8 emulator, used for running the complete system in a virtual environment.

This guide assumes only a **basic knowledge of Linux** and is designed for students and developers interested in embedded systems, virtualization, and bootloader development.

## Requirements

Ensure you have an Ubuntu 24.04 machine with the following dependencies installed:

```bash
sudo apt update
sudo apt install build-essential git autoconf automake libtool libexpat-dev \
    libncurses5-dev libgnutls28-dev bison flex patch gawk texinfo help2man \
    gperf python3-dev swig python3-setuptools python3-pyelftools libgmp-dev \
    libmpc-dev libmpfr-dev libisl-dev libssl-dev bc device-tree-compiler \
    parted qemu-system-arm qemu-user
```

The above installs a broad range of packages: compiler and build tools (build-essential, bison, flex, etc.), libraries needed by cross-compilation frameworks (GMP, MPC, MPFR for GCC, OpenSSL for certain firmware), the Device Tree Compiler (device-tree-compiler provides the dtc tool), QEMU for ARM/AArch64, and utilities like parted for disk partitioning. It also includes qemu-user which provides user-mode QEMU (useful for testing binaries with qemu-aarch64 interpreter). If you prefer to use a pre-built cross-compiler, you should also install the appropriate cross GCC (for example, gcc-aarch64-linux-gnu and related binutils). In our case, we will build a cross-compiler from source to demonstrate the process.

You will also need at least **15GB of free disk space** for compiling the various components. After the compilation, you can reclaim most of the space by issuing clean command in various directories.

## Repository Structure

```
linux-qemu-trusted-firmware/
│── docs/                  # Documentation and images
│── scripts/               # Helper scripts for setup
│── configs/               # Config files for Kernel, U-Boot, QEMU
│── rootfs/                # Sample root filesystem
│── LICENSE                # Open-source license (MIT, Apache, etc.)
│── README.md              # Main guide
│── setup.sh               # Optional script to automate setup
```

## Step-by-Step Guide

### 1. Build the Cross-Compilation Toolchain

We will use **crosstool-ng** to generate an ARM64 cross-toolchain.

To compile code for **ARMv8 (AArch64)** on an x86_64 host, we need a cross-compiler. While distribution packages provide ready-to-use toolchains (e.g., the `aarch64-linux-gnu-gcc` compiler), building a custom toolchain can give us control over libc (glibc vs musl), kernel header versions, and debugging tools. Here we use **Crosstool-NG**, a versatile toolchain builder.

First, create a working directory for the lab (this could be under /opt or your home directory). In this example, we use /opt/emblin/armv8-lab as a base directory and prepare a subdirectory for storing tarballs (source packages):

```bash
mkdir -p /opt/emblin/armv8-lab/tarballs  
cd /opt/emblin/armv8-lab/tarballs
```

Now clone the Crosstool-NG repository and checkout a known stable release:

```bash
git clone https://github.com/crosstool-ng/crosstool-ng.git
cd crosstool-ng
git checkout crosstool-ng-1.27.0
```

Next, build the Crosstool-NG tool itself:

```bash
./bootstrap    # Prepare the build system  
./configure --enable-local  # Configure for local (in-tree) installation  
make -j$(nproc)   # Build Crosstool-NG
```

At this point, the `ct-ng` binary should be built in the current directory (since we used `--enable-local`). We can list available sample configurations:

```bash
./ct-ng list-samples | grep rpi3
```

Crosstool-NG comes with sample configurations for various targets. Here we filter for "rpi3" (Raspberry Pi 3) samples, which target ARM Cortex-A53 (ARMv8). Suppose we find a sample named aarch64-rpi3-linux-gnu. We use it as a base:

```bash
./ct-ng aarch64-rpi3-linux-gnu  
```

This sets up a `.config` for building an AArch64 (64-bit ARM) Linux toolchain for Raspberry Pi 3. Before building, we customize the configuration. Run menuconfig:

```bash
./ct-ng menuconfig
```

In the menuconfig interface, we adjust some options:

- Under "Path and misc options", set the local tarball directory to /opt/emblin/armv8-lab/tarballs. This caches source tarballs (Linux kernel headers, GCC, Binutils, etc.) in our directory to avoid re-downloading.
- Under "Toolchain options", you may set a shorter tuple’s alias for convenience. For example, change the tuple alias to aarch64-linux (so the cross tools will be named e.g. aarch64-linux-gcc).
- Under "Operating System", select a Linux kernel header version. We choose 6.0.12 for a modern kernel headers.
- Under "C-library", select musl instead of glibc. Musl is a lightweight libc which simplifies our static linking later (BusyBox) and avoids large glibc dependencies in the target.
- Under "C compiler", select GCC version. For instance, GCC 13.3.0.
- (Optional) Under "Debug Facilities", enable inclusion of gdb and strace in the toolchain, if you want to have those debugging tools available.

After configuring, save and exit. Now build the toolchain (this process can take several minutes):

```bash
./ct-ng build
```

If the build succeeds, Crosstool-NG will install the cross-toolchain under your home directory by default (e.g., in ~/x-tools/aarch64-rpi3-linux-musl/ for our configuration). It also generates a script to set environment variables. In our example, an env.sh might be created to export the new PATH and other variables. We can verify and source it:

```bash
cd ..  # go back to /opt/emblin/armv8-lab
cat env.sh
```

This env.sh should contain lines like:

```bash
export PATH=${HOME}/x-tools/aarch64-rpi3-linux-musl/bin:${PATH}  
export CROSS_COMPILE=aarch64-linux-  
export ARCH=arm64
```

We load these variables into the current shell:

```bash
source env.sh
```

Now the cross-compiler aarch64-linux-gcc (and related tools such as aarch64-linux-ld, etc.) should be in PATH. A quick test: create a simple C file (hello.c) that prints "Hello world" and compile it:

```C
#include <stdio.h>

int main(void) {
   printf("Hello world\n");
   return 0;
}
```

Compile with the cross-compiler and run it under QEMU user-mode:

```bash
aarch64-linux-gcc -o hello-arm64 hello.c   # Cross-compile for AArch64  
qemu-aarch64 ./hello-arm64                # Use QEMU to run the binary
```

Very likely, you will get an runtime erro message du to the lack og the loader and the libc. You need to specify the dynamic linker path for QEMU. For example, if using musl, find the musl loader in the sysroot and run:

```bash
qemu-aarch64 -L ~/x-tools/aarch64-rpi3-linux-musl/aarch64-rpi3-linux-musl/sysroot ./hello-arm64
Hello world
```

This points QEMU to the cross-compiler’s sysroot for shared libraries. (For statically linked binaries, this is not needed.)

**Note:* Using **Crosstool-NG** is one way to obtain a cross-toolchain. Alternatively, one could install the package `gcc-aarch64-linux-gnu` which provides the cross-compiler and skip the above steps. In either case, ensure `CROSS_COMPILE` (prefix for toolchain binaries) is set (e.g., `aarch64-linux-`) and `ARCH=arm64` is used for Linux builds.

### 2. Build Trusted Firmware-A (TF-A)

```bash
git clone https://git.trustedfirmware.org/TF-A/trusted-firmware-a.git
cd trusted-firmware-a
git checkout v2.12.0
make PLAT=qemu ARCH=aarch64 DEBUG=1 CROSS_COMPILE=aarch64-unknown-linux-gnu- BL33=../u-boot/u-boot.bin all fip
```

### 3. Build U-Boot Bootloader

```bash
git clone https://github.com/u-boot/u-boot.git
cd u-boot
git checkout v2025.01
make CROSS_COMPILE=aarch64-unknown-linux-gnu- qemu_arm64_defconfig
make CROSS_COMPILE=aarch64-unknown-linux-gnu- -j$(nproc)
```

### 4. Compile the Linux Kernel

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux
git checkout v6.1.128
make ARCH=arm64 CROSS_COMPILE=aarch64-unknown-linux-gnu- defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-unknown-linux-gnu- -j$(nproc) Image dtbs modules
```

### 5. Prepare the Root Filesystem

```bash
git clone git://busybox.net/busybox.git
cd busybox
git checkout 1_36_0
make defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-unknown-linux-gnu- install
```

Create the root filesystem:

```bash
mkdir -p rootfs/{bin,sbin,etc,proc,sys,usr/bin,usr/sbin}
cp -a _install/* rootfs/
```

Generate the root filesystem image:

```bash
dd if=/dev/zero of=rootfs.img bs=1M count=128
mkfs.ext4 rootfs.img
mkdir mnt
sudo mount rootfs.img mnt
sudo cp -a rootfs/* mnt/
sudo umount mnt
```

### 6. Prepare the Boot Image

```bash
dd if=trusted-firmware-a/build/qemu/debug/bl1.bin of=flash.bin bs=4096 conv=notrunc
dd if=trusted-firmware-a/build/qemu/debug/fip.bin of=flash.bin seek=64 bs=4096 conv=notrunc
```

### 7. Run QEMU

```bash
qemu-system-aarch64 -machine virt,secure=on -cpu cortex-a53 -m 1024 -smp 2 -nographic -serial mon:stdio \
-bios trusted-firmware-a/build/qemu/debug/flash.bin -device virtio-blk-device,drive=hd0 \
-drive if=none,id=hd0,file=rootfs.img,format=raw
```

You should now see the **U-Boot** prompt, from which you can boot Linux manually.

## Booting Linux from U-Boot

```bash
setenv bootargs "console=ttyAMA0 root=/dev/vda rw"
fatload virtio 0:1 ${kernel_addr_r} Image
fatload virtio 0:1 ${fdt_addr} virt-smc.dtb
booti ${kernel_addr_r} - ${fdt_addr}
```

This should successfully boot Linux and drop you into a BusyBox shell.

## Troubleshooting

### Kernel Panics: Unable to Mount Root Filesystem

Ensure your kernel has **VirtIO Block Device Support** enabled:

```bash
make menuconfig
-> Device Drivers
    -> Block devices
        -> <*> VirtIO block driver
```

### U-Boot Fails to Load Kernel

Double-check that the kernel and DTB are present in the boot partition:

```bash
ls /mnt/boot/
```

### QEMU Serial Output Not Showing Kernel Logs

Try appending `earlyprintk=serial` to `bootargs` in U-Boot:

```bash
setenv bootargs "console=ttyAMA0 root=/dev/vda rw earlyprintk=serial"
```

## References

- [Trusted Firmware-A Documentation](https://trustedfirmware-a.readthedocs.io/)
- [U-Boot Documentation](https://u-boot.readthedocs.io/)
- [QEMU Documentation](https://www.qemu.org/docs/)
- [Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/)

## License

This project is licensed under the MIT License.


