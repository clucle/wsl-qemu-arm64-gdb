## Environment
 - Linux 6.6.36.3-microsoft-standard-WSL2 

## Install & Run

Install qemu

```bash
sudo apt-get update
sudo apt install qemu-system qemu-utils qemu-system-arm

# arm support machine list
qemu-system-arm --machine help
```

Build kernal image

```bash
git clone https://kernel.googlesource.com/pub/scm/linux/kernel/git/torvalds/linux

cd linux
git checkout v6.13

sudo apt-get install build-essential flex bison libncurses-dev bc libelf-dev libssl-dev gcc g++ zlib1g-dev gcc-aarch64-linux-gnu

make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig

# kernel hacking
# |--Compile-time checks and compiler options
# |   `--Debug information
# |       `--Rely on the toolchain's implicit default DWARF version
# |--Generic Kernel Debugging Instruments
# |   `-- [*] KGDB: kernel debugger
# |
# Kernel Features
# |--[ ] Randomize the address of the kernel image (KASLR) 

make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
```

Build initramfs image

```bash
git clone git://busybox.net/busybox.git
cd busybox

make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
# Settings -> [*] Build static binary (no shared libs)

make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- install

cd _install
mkdir -p dev etc/init.d home/root lib mnt proc root sys tmp usr/lib var

vi init

#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev
exec /bin/sh

chmod +x init

find . | cpio -H newc -o > ../../initramfs.cpio
```

Run image

```bash
qemu-system-aarch64 \
  -M virt \
  -cpu cortex-a57 \
  -smp 1 \
  -m 64 \
  -nographic \
  -kernel ./linux/arch/arm64/boot/Image  \
  -initrd ./initramfs.cpio \
  -append "console=ttyAMA0 earlycon rdinit=/init" \
  -device virtio-scsi-device \
  -d guest_errors
```

## Debug

Install debugger

```bash
sudo apt install gdb-multiarch
```

Run image with debug option
```bash
qemu-system-aarch64 \
  -M virt \
  -cpu cortex-a57 \
  -smp 1 \
  -m 64 \
  -nographic \
  -kernel ./linux/arch/arm64/boot/Image  \
  -initrd ./initramfs.cpio \
  -append "console=ttyAMA0 earlycon rdinit=/init" \
  -device virtio-scsi-device \
  -d guest_errors \
  -gdb tcp::1234 \
  -S
```

Debug

```bash
gdb-multiarch ./linux/vmlinux
(gdb) break start_kernel
Breakpoint 1 at 0xffff800081ce08d4: file init/main.c, line 905.
(gdb) target remote localhost:1234
0x0000000040000000 in ?? ()
(gdb) continue
Continuing.

Breakpoint 1, start_kernel () at init/main.c:905
905             set_task_stack_end_magic(&init_task);
```
