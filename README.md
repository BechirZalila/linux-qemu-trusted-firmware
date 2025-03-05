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
- Under "Toolchain options", you may set a shorter tuple's alias for convenience. For example, change the tuple alias to aarch64-linux (so the cross tools will be named e.g. aarch64-linux-gcc).
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

This `env.sh` should contain lines like:

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

This points QEMU to the cross-compiler's sysroot for shared libraries. (For statically linked binaries, this is not needed.)

**Note:** Using **Crosstool-NG** is one way to obtain a cross-toolchain. Alternatively, one could install the package `gcc-aarch64-linux-gnu` which provides the cross-compiler and skip the above steps. In either case, ensure `CROSS_COMPILE` (prefix for toolchain binaries) is set (e.g., `aarch64-linux-`) and `ARCH=arm64` is used for Linux builds.

To clean up the temporary file space used bu crosstool-ng you may do the followinig:

```bash
cd crosstool-ng
./ct-ng clean
```

We will now obtain the source code for the components we need: TF-A, U-Boot, Linux kernel, and BusyBox:
- ARM Trusted Firmware-A (TF-A): We'll use the official Arm repository.
- U-Boot Bootloader: We use a recent stable release from Denx.
- Linux Kernel: We will use a long-term support kernel (6.1.x series).
- BusyBox: We will use the latest stable BusyBox for user-space.

We will download these as needed in the following sections.

### 2. Build Trusted Firmware-A (TF-A)

Trusted Firmware-A provides the secure-world boot stages (BL1, BL2, BL31) for ARMv8. In QEMU's `virt` machine, TF-A's BL1 is the first code to run at reset, and it is loaded at the reset vector (0x0 in secure memory) via the QEMU `-bios` option. BL1 will initialize EL3, choose a primary CPU, and eventually load BL2. BL2, in turn, will load BL31 (and any optional BL32 secure payload) and the BL33 image into memory. In our setup, we will not use a BL32 (secure OS like OP-TEE) for simplicity, and will use U-Boot as BL33 (non-secure firmware). BL2 also prepares the device tree for the next stage: on QEMU, BL2 patches the DTB to add a PSCI node and set the CPU enable methods (how secondary CPUs are brought up). This ensures the Linux kernel (or U-Boot) gets a correct device tree with PSCI information.

We obtain the TF-A source and build it for the QEMU platform:

```bash
git clone https://github.com/ARM-software/arm-trusted-firmware.git
cd arm-trusted-firmware
git checkout v2.12.0
```

Now compile TF-A for QEMU:

```bash
make PLAT=qemu ARCH=aarch64 CROSS_COMPILE=${CROSS_COMPILE} DEBUG=1
```

Let's break down this command: `PLAT=qemu` selects the QEMU virt platform support in TF-A, `ARCH=aarch64` sets 64-bit, and `CROSS_COMPILE=${CROSS_COMPILE}` uses our cross-toolchain prefix. `DEBUG=1` builds a debug version with additional log messages (useful for development; for a release build you might omit this). This initial build will produce several binaries under `build/qemu/debug/` (or `release/` if `DEBUG=0`): notably `bl1.bin`, `bl2.bin`, `bl31.bin`, and a file `fip.bin` (Firmware Image Package) if configured to produce one. However, because we did not specify a BL33 at this time, the fip may not contain a BL33 payload yet. We will address that after building U-Boot. For now, we have built the core TF-A components.

(Optional) Understanding TF-A output: The BL1 is a standalone binary that will reside at the start of the flash. The FIP is a packaged binary that contains BL2, BL31, and optionally other images like BL33 and BL32. QEMU's secure flash is organized such that BL1 occupies offset 0x0 up to 256 KB, and the FIP is placed after that offset. The size of BL1 is small (on the order of 20 KB), but QEMU expects the BL1 region to be 256 KB; the FIP follows it. We will later concatenate `bl1.bin` and `fip.bin` to create a single flash image.

Before proceeding, confirm that TF-A built correctly. The build artifacts will be in `build/qemu/debug/` (since we did a debug build). We will come back to TF-A to integrate U-Boot once U-Boot is ready, so keep this directory in mind.

**Important Build Parameters:** We specified `PLAT=qemu` which picks default settings for the QEMU virt platform. TF-A's platform support for QEMU knows the memory map of the `virt` machine, including device addresses, flash size, and the requirement to concatenate BL1 and FIP. (In contrast, for a real board, PLAT might be something like `raspi4` or `fvp` etc., with different addresses.) The `CROSS_COMPILE` must point to an AArch64 bare-metal GCC for building TF-A, since TF-A is freestanding (no Linux libc). Our cross toolchain is built for Linux user-space, but it works for building TF-A because it can compile freestanding code. Alternatively, one could use a dedicated AArch64 ELF toolchain. In practice, aarch64-linux-gcc works for TF-A. Setting `DEBUG=1` enables assertions and verbose logs in TF-A; for a production build use `DEBUG=0` (default). If building a release, append `PLAT=qemu` `SPD=none` to explicitly indicate no secure payload (since we aren't using BL32 in this tutorial).

### 3. Build U-Boot Bootloader

U-Boot will be our Stage-2/Stage-3 bootloader (BL33) running in Normal World. After TF-A's BL31 passes control to BL33, U-Boot will execute in EL2 (if available) or EL1, set up devices, and ultimately launch the Linux kernel. We will configure U-Boot for QEMU's virt board and ensure it aligns with TF-A's expectations (e.g., load address).

First, obtain U-Boot. Here we download a stable release archive (`2025.01` as an example):

```bash
wget -c https://ftp.denx.de/pub/u-boot/u-boot-2025.01.tar.bz2
tar xf u-boot-2025.01.tar.bz2
mv u-boot-2025.01 u-boot
cd u-boot
```

Now configure U-Boot for the QEMU ARM64 virtual board:

```bash
make qemu_arm64_defconfig
```

The `qemu_arm64_defconfig` is a predefined configuration for QEMU's AArch64 virt machine. This defconfig sets up U-Boot to run as a firmware on the virt board. By default, U-Boot expects to be loaded at address `0x40000000` (the start of DRAM on many ARM boards). However, in our case TF-A will load BL33 at `0x60000000`. We need to adjust U-Boot's link address (`TEXT_BASE`) accordingly so that U-Boot knows its starting address. We can do this via U-Boot's menuconfig:

```bash
make menuconfig
```

In the menu, navigate to **"General Setup" -> "Text Base"** and set it to `0x60000000`. This will define `CONFIG_TEXT_BASE=0x60000000`, meaning U-Boot will be linked to run from that address. (This address is chosen because TF-A's BL2 will load BL33 there in memory. The TF-A QEMU platform code uses `0x60000000` for BL33 entry point, as also indicated by TF-A's boot messages or documentation.)

Next, we configure U-Boot's environment storage. By default, U-Boot may expect to store its environment (persistent variables) in non-volatile storage (SPI flash or NAND) or not at all. On QEMU virt, we don't have real flash accessible from U-Boot. A convenient method is to store the U-Boot environment in a file on the boot disk (e.g., on the FAT partition). U-Boot has a feature to save environment to a FAT filesystem. We enable this in menuconfig:

Navigate to **"Environment"** settings. Disable **"Environment in flash memory"** (uncheck if set, by hittting the space bar). Enable **"Environment is in a FAT filesystem"**. Then set the interface to **"virtio"** (because our disk will appear as a virtio-blk device in QEMU) and set **"Device and partition"** to `0:1` (meaning first virtio disk, first partition). Set the **"Name of the FAT file"** to `uboot.env` (this is the file that will store environment variables).

These settings correspond to `CONFIG_ENV_IS_IN_FAT=y`, `CONFIG_ENV_FAT_INTERFACE="virtio"`, `CONFIG_ENV_FAT_DEVICE_AND_PART="0:1"`, and `CONFIG_ENV_FAT_FILE="uboot.env"` in U-Boot's configuration. In effect, U-Boot will, on a `saveenv` command, write its environment to `uboot.env` on the first partition of the virtio block device, and load from there on boot if available. This provides persistency of U-Boot environment across reboots inside our QEMU setup, much like a real board's NVRAM or flash.

After configuring, exit and save the changes. Now build U-Boot:

```bash
make -j$(nproc)
```

This compiles U-Boot using the cross-compiler (ensure `CROSS_COMPILE` is still set in your environment, e.g., from `env.sh`). If successful, it produces several binaries; the main ones are `u-boot` (ELF format, useful for debugging), and `u-boot.bin` (binary image). The U-Boot binary will be placed at our configured text base when executed. We can verify the embedded entry point:

```bash
aarch64-linux-readelf -h u-boot
```

Look for **"Entry point address"** in the ELF header. It should read `0x60000000`, confirming that U-Boot will start at `0x60000000`. If it shows `0x40000000`, then the text base config did not take effect – re-check the menuconfig step.

U-Boot is now ready, but we need to integrate it with TF-A so that it will be loaded as BL33. The next step will combine TF-A (BL1, BL2, BL31) and U-Boot (BL33) into a single flash image. This will be done in the next section.

### 4. Creating a Disk Image and Partitions

We create an empty image file of size 256 MB to simulate an SD card or eMMC. Then we partition it into two: a FAT32 boot partition and an `ext4` root partition.

```bash
dd if=/dev/zero of=disk.img bs=1M count=256
```

This creates a 256 MB file filled with zeros. The `dd` command uses a block size of 1MiB and writes 256 blocks (resulting in 256 MiB). Next, partition the image with parted:

```bash
parted disk.img --script mklabel msdos  
parted disk.img --script mkpart primary fat32 1MiB 100MiB  
parted disk.img --script mkpart primary ext4 100MiB 100%
```

**Explanation:** `mklabel` msdos creates a Master Boot Record (MBR) partition table (also known as MS-DOS label) on the image. We choose MBR for simplicity (GPT could also be used, but U-Boot and our needs are fine with MBR). Then we create two primary partitions. The first partition is of type FAT32, starting at 1MiB and ending at 100MiB. We start at 1MiB instead of 0 to align the partition and leave space for the MBR and potential bootloader metadata; 1MiB alignment is a common practice to ensure proper alignment for performance. The size of the first partition will be roughly 99 MB (from 1 to 100). The second partition is from 100MiB to 100% (end of the disk), occupying the rest (~156 MB) and will be ext4. After these commands, the layout is:
- Partition 1 (vda1): ~99 MB FAT32, for boot files (kernel Image, device tree, and U-Boot environment file).
- Partition 2 (vda2): ~156 MB ext4, for the root filesystem (BusyBox and others).

Now we format these partitions with the appropriate filesystems. Because disk.img is not a real device, we use a loopback device. We can have Linux map the image as a loop device and create sub-devices for its partitions:

```bash
sudo losetup -fP --show disk.img
```

The `-fP` options find the first free loop device and partition the device, respectively. The `--show` prints the loop device name. Suppose it returns `/dev/loop0`. The `-P` flag causes the kernel to scan the partition table of `/dev/loop0` and create `/dev/loop0p1` and `/dev/loop0p2` for the partitions. We can confirm with `lsblk` or `sudo fdisk -l /dev/loop0`.

Now format the partitions:

```bash
sudo mkfs.vfat -F 32 /dev/loop0p1  
sudo mkfs.ext4 -L ROOT /dev/loop0p2
```

We make the first partition a FAT32 filesystem (`-F 32` for FAT32) and the second an `ext4` filesystem, labeling it "ROOT" (this label is optional, but can help identify the partition).

After formatting, detach the loop device for now:

```bash
sudo losetup -d /dev/loop0
```

We now have a disk.img with two empty filesystems. Next, we will populate them: the FAT32 partition with the U-Boot environment file, the kernel and device tree, and the ext4 partition with a minimal root filesystem.

### 5. Creating the Flash Image (BL1 + FIP)

Recall that QEMU's virt platform expects a flash image where BL1 is at offset 0 and the FIP (containing BL2, BL31, BL33) is at offset `0x40000` (256 KB). We will use the TF-A tool fip to bundle BL2, BL31, and BL33, and then concatenate BL1.

Go back to the TF-A directory:

```bash
cd arm-trusted-firmware
```

We will rebuild TF-A with U-Boot included as BL33. Clean the previous build and run make with BL33 specified:

```bash
make distclean  
make PLAT=qemu ARCH=aarch64 CROSS_COMPILE=${CROSS_COMPILE} DEBUG=1 \
     BL33=../u-boot/u-boot.bin all fip
```

This command does a full rebuild (`all`) and creates a firmware image package (`fip`). By specifying `BL33=../u-boot/u-boot.bin`, we include the U-Boot binary as the BL33 payload in the FIP. After this, in `build/qemu/debug/` we will have a new `fip.bin` alongside a new `bl1.bin` (and other BL images). The FIP contains BL2, BL31, BL33 = U-Boot (and potentially empty slots for BL32 since we didn't specify one).

Now we create a single flash image file. We need to allocate space for BL1 up to 256KB, then the FIP after that. A simple and reliable method is using `dd` to write `bl1.bin` into the beginning of a file and `fip.bin` at the 256KB offset:

```bash
cd build/qemu/debug  
dd if=bl1.bin of=flash.bin bs=4096 conv=notrunc  
dd if=fip.bin of=flash.bin bs=4096 seek=64 conv=notrunc
```

The first `dd` writes BL1 into `flash.bin` with a block size of 4096 bytes (4KB) and `conv=notrunc` to not truncate the output file. The second dd writes `fip.bin` at an offset: `seek=64` means skip 64 blocks of 4096 bytes = 256KB. This ensures `flash.bin` has BL1 starting at 0 and FIP starting at `0x40000`. The size of `flash.bin` will be at least (256KB + size of `fip.bin`). Typically, TF-A's documentation suggests flash size around 4MB for QEMU `virt`, which is sufficient for our BL1+FIP. The exact size isn't critical as long as both pieces fit. Now `flash.bin` is the firmware image we will give to QEMU as the `-bios`.

We can now launch QEMU, combining everything. We must be at the top level work directory in order for the QEMU command to work.

```bash
qemu-system-aarch64 -machine virt,secure=on -cpu cortex-a53 -m 1024 \
   -smp 2 -nographic -serial mon:stdio \
   -bios arm-trusted-firmware/build/qemu/debug/flash.bin \
   -device virtio-blk-device,drive=hd0 \
   -drive if=none,id=hd0,file=disk.img,format=raw
```

Let's decode these options:
- `-machine virt,secure=on`: Select the "virt" generic ARMv8 board in secure mode. `secure=on` enables two worlds (Secure and Normal), which is necessary for TF-A to run (it will run in Secure EL3). This also means QEMU will boot from the provided secure firmware (`flash.bin`).
- `-cpu cortex-a53`: Emulate a Cortex-A53 CPU (ARMv8 little core). This CPU has EL3 support and is compatible with our use of TrustZone. Cortex-A53 is a reasonable choice (Cortex-A57 or others could be used; A53 is fine and was used in Raspberry Pi 3 for example).
- `-m 1024 -smp 2`: Allocate 1024 MB of RAM and 2 CPU cores to the VM. QEMU will reflect these in the generated device tree (memory node size `0x40000000` for 1GB, and two CPU nodes) which should match our earlier device tree assumptions.
- `-nographic`: Run with no graphical output (no GUI), and redirect serial to the console.
- `-serial mon:stdio`: This multiplexes the QEMU monitor and the serial console onto QEMU's standard input/output. Upon startup, you see the serial output; to switch to the QEMU monitor (to issue QEMU commands), one typically presses Ctrl-A C when using `mon:stdio`. This setup allows us to interact with U-Boot's serial console via the terminal.
- `-bios flash.bin`: Provide the binary firmware to load at reset (instead of QEMU's default BIOS or UEFI). This `flash.bin` is loaded at address `0x0` in the secure memory. QEMU will start executing at the reset vector, which is in BL1 inside our `flash.bin`. From there TF-A takes over the early boot.
- `-device virtio-blk-device,drive=hd0` `-drive if=none,id=hd0,file=disk.img,format=raw`: This attaches our `disk.img` as a virtio block device. The `-drive if=none,id=hd0,...` defines a drive backing file, and `-device virtio-blk-device,drive=hd0` exposes it to the guest as a virtio-blk PCI device. In the guest Linux, this will appear as `/dev/vda`. U-Boot will also see a `virtio 0` device representing this disk. We use virtio-blk because it's a simple para-virtualized storage device suited for QEMU; it will use the Virtio driver.

If all is set up correctly, running this command should produce output on the console. Let's walk through the expected boot sequence:
- TF-A BL1 runs first (very briefly). With `DEBUG=1`, you'll see some TF-A log messages like:
  `NOTICE: BL1: v2.12(release):` ... and then messages about loading BL2, etc. BL1 will load BL2 (from FIP).
- TF-A BL2 executes. BL2 will load BL31 and U-Boot (BL33). You may see logs such as:
  `NOTICE: BL2: Loading BL31` and `NOTICE: BL2: Loading BL33` with addresses. BL2 also might print a notice about transferring control to BL31.
- TF-A BL31 executes (EL3 runtime). It will initialize and then drop to the non-secure world by invoking BL33 at `0x60000000`. We might see a `NOTICE: BL31: ...` and then perhaps PSCI init messages. BL31 uses SMC/HVC for PSCI as needed. At this point, control goes to U-Boot.
- U-Boot (BL33) starts. You should see the familiar U-Boot banner on the serial console: e.g.,
  `U-Boot 2025.01 [...]` and a message about DRAM, etc., followed by the U-Boot command prompt =>. If U-Boot output doesn't appear, something is wrong (check TF-A logs or U-Boot integration). If it appears, we have a working TF-A + U-Boot stack.

At this level, U-Boot is ready to boot the Linux kernel, which will be built in the next section. 

### 6. Compile the Linux Kernel

For our ARMv8 Linux system, we use the mainline Linux kernel. Here we choose version `6.1.128` (a stable LTS release in the `6.1` series). Obtain and extract the kernel source:

```bash
wget -c https://mirrors.edge.kernel.org/pub/linux/kernel/v6.x/linux-6.1.128.tar.xz
tar xf linux-6.1.128.tar.xz
mv linux-6.1.128 linux
cd linux
```

Now configure the kernel for the virt platform. The arm64 architecture kernel has a default configuration that includes support for generic devices:

```bash
make defconfig   # generates a default arm64 kernel config
```

The default config (defconfig) for arm64 should enable drivers for the PL011 serial (used for ttyAMA0) and virtio devices, which are crucial for QEMU's virt machine. It typically also enables `devtmpfs` and other basic features. However, it might not include every feature we need. We can customize it with menuconfig:

```bash
make menuconfig
```

For our simple system, we don't need to change much beyond ensuring virtio and ext4 support are built-in. If in doubt, the default config on arm64 is generally sufficient for booting on QEMU virt. After any changes, save the config.Now build the kernel image:

```bash
make -j$(nproc) Image
```

This compiles the Linux kernel and produces an uncompressed binary image at `arch/arm64/boot/Image`. (We use Image rather than `Image.gz` to keep it simple – U-Boot can boot the uncompressed Image, and we built U-Boot's `booti` command to handle both compressed and uncompressed.) If the build succeeds, `arch/arm64/boot/Image` is our kernel binary.

**Device Tree Considerations:** The Linux kernel on ARM typically requires a Device Tree Blob (DTB) to describe the hardware. For QEMU’s virt board, a device tree is automatically generated by QEMU at runtime and passed to the bootloader or kernel. In our multi-stage boot flow, we have two choices:
1. Let QEMU provide the device tree directly to TF-A/U-Boot and have U-Boot pass it to the kernel.
2. Extract the QEMU-generated device tree and use it as an input file that U-Boot can load and pass to the kernel.

To avoid any uncertainty, we will use approach (2): dump the QEMU device tree and compile it into a DTB that we explicitly load. This also gives us a chance to modify the device tree if needed.

Obtaining the QEMU device tree: QEMU provides a convenient option to dump the device tree of the `virt` machine without booting an OS. We can use:

```bash
qemu-system-aarch64 -machine virt,secure=on -machine dumpdtb=qemu.dtb -cpu cortex-a53 -m 1024
```

This command will create a file `qemu.dtb` containing the device tree for a virt machine with the given parameters (we include `secure=on` because that slightly affects the DTB by adding secure firmware nodes). We should include any devices we plan to attach (like virtio-blk) in this dump command, because the DTB will only list devices that are present. For example:

```bash
qemu-system-aarch64 -machine virt,secure=on,dumpdtb=qemu.dtb -cpu cortex-a53 -m 1024 \
  -device virtio-blk-device,drive=hd0 -drive if=none,id=hd0,file=disk.img,format=raw
```

Alternatively, one can retrieve the DTB at runtime via U-Boot: start QEMU with U-Boot and use U-Boot’s fdt commands to print the device tree QEMU passed. For example, in U-Boot:

```bash
=> fdt addr <address>
=> fdt print
```

where <address> is where the device tree is located in memory. In practice, one can find it by searching memory or by launching the QEMU command with `-kernel ..../Image` and see from where it reads the DTB (we found it at `0x7e682d10`). However, using `-dumpdtb` is straightforward and does not require any guessing.

In all cases the found DTB must be decompiled and then recompiled again:

```bash
dtc -I dtb -O dts -o virt.dts qemu.dtb
```

Open virt.dts in an editor and find the psci node. It should look like:

```c
psci {  
    compatible = "arm,psci-1.0", "arm,psci-0.2";  
    method = "smc";  
    ...  
};  
```

If method is `"hvc"`, change it to `"smc"`. Save the DTS. If other adjustments are needed (for example, ensure the memory size matches the -m 1024 we will use, and CPU count matches -smp 2), check those as well: the `/memory` node reg should be `0x40000000` with size `0x40000000` (1GB) for 1024 MB, and the `/cpus` node should have 2 CPU subnodes for smp 2. QEMU’s dumpdtb should already reflect these parameters if provided.

Now compile the DTS back to DTB:

```bash
dtc -I dts -O dtb -o virt.dtb virt.dts
```

We now have virt.dtb, the device tree blob that the kernel will use. We must place the kernel `Image` and device tree in the boot partition (FAT32) so U-Boot can load them. Mount the FAT partition and copy:

```bash
mkdir -p part1  
sudo losetup -fP --show disk.img   # attach again (loop0)  
sudo mount /dev/loop0p1 part1  
sudo cp linux/arch/arm64/boot/Image virt.dtb part1/  
sudo umount part1  
sudo losetup -d /dev/loop0
```

Finally, we launch QEMU and check that the kernel boot successfully:

```bash
qemu-system-aarch64 -machine virt,secure=on -cpu cortex-a53 -m 1024 -smp 2 \
    -nographic -serial mon:stdio -bios arm-trusted-firmware/build/qemu/debug/flash.bin \
    -device virtio-blk-device,drive=hd0 -drive if=none,id=hd0,file=disk.img,format=raw
```

At the U-Boot prompt, we need to boot the Linux kernel. We have copied the kernel Image and device tree to the FAT partition of the disk, which U-Boot can access via the virtio block driver. U-Boot should have a driver for virtio-blk (since QEMU’s “virt” is a common target, U-Boot includes virtio support). The disk will appear to U-Boot as a block device accessible via the virtio interface. We can confirm by listing files on the first partition:

```bash
=> fatls virtio 0:1
```

This means: list directory of interface virtio drive 0, partition 1. It should show Image and virt.dtb (and possibly uboot.env if it exists). If this fails, it might mean U-Boot did not detect the drive; check that CONFIG_VIRTIO_BLK=y was in U-Boot config (it should be by default for qemu_arm64). If the files are listed, proceed to load them:

```bash
=> fatload virtio 0:1 ${kernel_addr_r} Image
=> fatload virtio 0:1 ${fdt_addr} virt.dtb
```

These commands load the kernel and device tree into RAM. ${kernel_addr_r} and ${fdt_addr} are environment variables set by U-Boot’s default environment (they point to safe load addresses in RAM, typically 0x40400000 or similar for kernel, and another address for fdt). We can check printenv kernel_addr_r to see the actual address. Make sure these addresses do not overlap. Usually, kernel_addr_r is something like 0x40800000 and fdt_addr like 0x4f000000, which in 1GB RAM are fine. Now set the kernel boot arguments:

```bash
=> setenv bootargs "console=ttyAMA0 root=/dev/vda2 rw"
```

This sets the environment variable bootargs that the Linux kernel will receive from U-Boot. We use the parameters discussed earlier. Now boot the kernel using U-Boot’s booti command (for AArch64 kernels):

`console=ttyAMA0` directs kernel logs to the first PL011 UART (which QEMU maps to the host’s stdio). This is essential to see output. `root=/dev/vda2` tells the kernel to use the second partition of the virtio disk as the root filesystem. `/dev/vda` is the typical name of the first virtio block device in Linux (with partitions vda1, vda2, etc.). We will put our rootfs on vda2. The rw flag mounts it read-write. We could also specify `rootfstype=ext4` to be explicit, but the kernel can auto-detect the filesystem type. If the root device might not be ready immediately (not an issue for virtio, but for safety on real hardware one might add `rootdelay=1`), we could add a short root delay. In our QEMU case, it should not be needed. We will set this bootargs in U-Boot later.

```bash
=> booti ${kernel_addr_r} - ${fdt_addr}
```

The booti syntax is booti <kernel_address> <initrd_address> <fdt_address> for booting an uncompressed Linux AArch64 kernel. We have no initrd, so we use - to indicate none. U-Boot should respond with messages like “Loading Device Tree to ...” and then “Starting kernel ...”. At that point, control is transferred to the Linux kernel.

Now the Linux kernel should boot. Early on, it will print messages via the serial console (which we’ve set to ttyAMA0). You should see the typical Linux boot log: the kernel version banner, information about the CPUs (it should detect 2 CPUs and bring them up using PSCI), memory, virtio devices (it should identify a virtio block device vda, and maybe entropy device etc.), then it will try to mount the root filesystem on /dev/vda2. At this stage, the kernel hangs at a Kernel Panic saying it cannot mount the root file system, this is normal since wi did not build it yet. 

**Saving the Boot Command (Optional):** Right now, we manually entered commands in U-Boot to boot Linux. To automate this for future boots, we can use U-Boot’s saveenv to persist the bootcmd. First, while still in U-Boot (before booti or if you reset and break into U-Boot again), set the bootcmd variable:

```bash
=> setenv bootargs "console=ttyAMA0 root=/dev/vda2 rw"
=> setenv bootcmd 'fatload virtio 0:1 ${kernel_addr_r} Image; fatload virtio 0:1 ${fdt_addr} virt-dump.dtb; booti ${kernel_addr_r} - ${fdt_addr}'
=> saveenv
```

This writes the environment to uboot.env on the FAT partition (since we configured that). Now, upon next boot, U-Boot will automatically execute bootcmd after a timeout, which will load the kernel and dtb and boot Linux without manual intervention. You can test this by resetting QEMU (if at the QEMU monitor, type reset, or simply exit and re-run QEMU). U-Boot should count down and then boot Linux.

### 5. Prepare the Root Filesystem

The Linux kernel by itself is not useful without a root filesystem. For the root filesystem, we use BusyBox, which provides a suite of Unix utilities and can serve as PID1 (init process). BusyBox can be built as a single static binary implementing essential commands (sh, ls, etc.), ideal for an embedded system.

Download and extract BusyBox (1.36.1 is used here):

```bash
wget -c "https://www.busybox.net/downloads/busybox-1.36.1.tar.bz2"
tar xf busybox-1.36.1.tar.bz2
cd busybox
```

Configure BusyBox:

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux- defconfig
```

This sets up a default configuration. The default BusyBox config is geared towards a shared-library build using glibc. We want a static binary (no external library dependencies) so that our rootfs does not need glibc or musl libraries. Enable static linking:

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
```

In BusyBox menuconfig, go to Settings and activate “Build static binary (no shared libs)” (this sets CONFIG_STATIC=y). You can also adjust other applet selections if desired, but the default set is fine for basic functionality. Save the config. Now build and install BusyBox:

```bash
make ARCH=arm64 CROSS_COMPILE=${CROSS_COMPILE} -j$(nproc)  
make ARCH=arm64 CROSS_COMPILE=${CROSS_COMPILE} install
```

The install step will create a _install directory in the BusyBox source tree, containing the files of a minimal filesystem. Let’s examine _install: it should have directories like bin, sbin, etc, usr/bin, and symlinks for busybox applets. BusyBox’s install target effectively performs make CONFIG_PREFIX=_install install, which populates a fake rootfs.

Now we will construct our actual root filesystem in the ext4 partition. Mount the ext4 partition and copy the files:

```bash
mkdir -p part2

sudo losetup -fP --show disk.img
sudo mount /dev/loop0p2 part2

cd part2
sudo cp -a ../busybox/_install/* .
```

The -a flag copies all files preserving permissions. This places busybox binary and symlinks into the root of our ext4 volume. Next, create necessary directories that BusyBox did not install by default:

```bash
sudo mkdir -p etc/init.d proc sys dev
```

We create /etc, /etc/init.d, /proc, /sys, and /dev in the new rootfs. proc and sys are for the procfs and sysfs which we will mount at boot. dev is for device nodes; modern Linux with devtmpfs will populate this at runtime if enabled. We also need an init script. BusyBox will look for an initialization script at /etc/init.d/rcS by default (because BusyBox’s init system follows a simple RC script approach if no /etc/inittab is provided). We create this rcS script:

```bash
sudo sh -c 'echo -e "#!/bin/sh\nmount -t proc none /proc\nmount -t sysfs none /sys" > etc/init.d/rcS'
```

Then make it executable:

```bash
sudo chmod a+x etc/init.d/rcS
```

The script will mount the proc filesystem and sysfs at boot. We do not explicitly mount devtmpfs here; the Linux kernel (if configured with CONFIG_DEVTMPFS_MOUNT) will automatically mount a tmpfs at /dev early on. If not, we might add mount -t devtmpfs none /dev to rcS, but most distribution kernels, including defconfig, mount devtmpfs for us. We keep rcS minimal: it’s essentially the startup sequence. After rcS, BusyBox’s init will typically spawn a shell on the console (because we have no inittab, BusyBox falls back to a single shell on ttyAMA0). This means when the system boots, we should get a shell prompt directly.

Now our root filesystem partition has BusyBox and basic directories. Unmount and detach the loop:

```bash
cd ..
sudo umount part2
sudo losetup -d /dev/loop0
```

The ext4 partition (rootfs) is ready.

We can now launch QEMU, combining everything:

```bash
qemu-system-aarch64 -machine virt,secure=on -cpu cortex-a53 -m 1024 -smp 2 \
    -nographic -serial mon:stdio -bios arm-trusted-firmware/build/qemu/debug/flash.bin \
    -device virtio-blk-device,drive=hd0 -drive if=none,id=hd0,file=disk.img,format=raw
```

Provided everything is correct, it will find the ext4 partition, mount it, and then invoke the init process. Since we are using BusyBox with no custom init system, BusyBox’s built-in init should run, execute our /etc/init.d/rcS. That script will mount proc and sysfs. After rcS completes, typically BusyBox will spawn a shell on the console (on ttyAMA0) as root. You should then see a prompt (likely # or similar). If you get a prompt, congratulations – you have a working Linux system!

At the shell, you can run some basic commands to test: ls / (should show directories like bin, sbin, etc, proc, sys, dev), uname -a (to see kernel version and architecture), dmesg | less (to review kernel messages, though less might not be available in our BusyBox, you can use dmesg alone). You can also verify that both CPUs are online (e.g., grep -i cpu /proc/cpuinfo should list 2 CPUs) and that the root filesystem is mounted read-write (mount command output).

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


