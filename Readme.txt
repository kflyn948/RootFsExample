
.
├── bin
├── dev
├── etc
├── home
├── lib
├── lib64
├── proc
├── sbin
├── sys
├── tmp
├── usr
│   ├── bin
│   ├── lib
│   └── sbin
└── var
    └── log

Filesystem Hierarchy Standard

Mastering Embedded Linux Programming Chapter 5
http://refspecs.linuxfoundation.org/fhs.shtml
● /bin - programs for all users, used at boot
● /dev - device nodes and other files
● /etc - system configuration files
● /lib - shared libraries
● /proc, /sys - proc and sysfs filesystem
● /sbin - programs for the system administrator, used at boot
● /tmp - temporary files - can be deleted on boot
● /usr /usr/bin, /usr/sbin - additional programs libraries, utilities
● /var - files modified at runtime (/var/log) which need to be retained after boot

How do we create a rootfs?

Step1: create root folder 

mkdir rootfs && cd rootfs

mkdir -p bin dev etc home lib lib64 proc sbin sys tmp usr var
mkdir -p usr/bin usr/lib usr/sbin
mkdir -p var/log

What about the files?
Mastering Embedded Linux Programming Chapter 5 12
● Now we’ve got the filesystem structure, where do we get the files we need to fill it?
● Use Busybox
● Single binary that implements essential Linux programs
● Symbolic links are used to tell which program is being requested

step2:Make and install busybox:
git://git.busybox.net/busybox
○ make distclean
○ make defconfig
○ make ARCH=${ARCH} CROSS_COMPILE=${CROSS_COMPILE}
○ make CONFIG_PREFIX=/path/to/rootdir ARCH=${ARCH} CROSS_COMPILE=${CROSS_COMPILE} install
example:
make ARCH=arm CROSS_COMPILE=arm-none-linux-gnueabi- defconfig
make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- defconfig



step3: Add needed shared libraries from toolchain sysroot( busybox needs some shared library and program interpreter)

  3.1 Use aarch64-none-linux-gnu-gcc as a cross toolchain example, get the sysroot path by running the command as below:

  aarch64-none-linux-gnu-gcc --print-sysroot

  3.2 find  out the dependcy of busybox using aarch64-none-linux-gnu-readelf and copy the file from sysrooot dir into lib r lib64

  aarch64-none-linux-gnu-readelf -a bin/busybox | grep "program interpreter"
  aarch64-none-linux-gnu-readelf -a bin/busybox | grep "Shared library"

step4: Devices Created with mknod (make node)

mknod <name> <type> <major> <minor>

○ Null device is a known major 1 minor 3
○ Console device is known major 5 minor 1

sudo mknod -m 666 dev/null c 1 3
sudo mknod -m 600 dev/console c 5 1

Make the contents owned by root:

cd rootfs
sudo chown -R root:root *


step5: Using your rootfs with the target
● Ramdisk
○ Disk image loaded in RAM by bootloader
● Disk Image
○ Disk image for use with memory (like sdcard)
● Network File System (NFS)


Use CPIO to create initramfs
find . | cpio -H newc -ov --owner root:root > ${OUTDIR}/initramfs.cpio
gzip -f initramfs.cpio

    


++++++++++++++++++++++++++++++++++++++++++++++++++
++++++++++++++++++building the kernel+++++++++++++

● Kconfig/menuconfig allows you to specify 3 modes for most drivers
○ off
■ Not built/included
○ on
■ built in to the kernel
○ Module
■ loadable at runtime


make menuconfig
● Changes are stored 
in a .config file at 
kernel source root


Compiling with Kbuild
Mastering Embedded Linux Programming Chapter 4 9
https://github.com/torvalds/linux/blob/v5.3-rc8/drivers/char/Makefile
https://github.com/torvalds/linux/blob/v5.3-rc8/Documentation/kbuild/makefiles.rst
● Example from drivers/char/Makefile
● obj-y = “built in”
● obj-m = “Module”
● needs mem.o and random.o built into the 
kernel (-y)
● ttyprintk may be included into the kernel or 
as a module based on 
CONFIG_TTY_PRINTK


Build Artifacts
Mastering Embedded Linux Programming Chapter 4 10
● vmlinux
○ ELF binary - possibly with debugging 
symbols
● Binaries in arch/$ARCH/boot for use with bootloaders
○ Image - binary version of vmlinux
○ zImage - compressed Image
○ uImage - compressed image with U-Boot header

● make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- mrproper
○ “deep clean” the kernel build tree - removing the .config file with any existing configurations

● make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- defconfig
○ Configure for our “virt” arm dev board we will simulate in QEMU

● make -j4 ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- all
○ Build a kernel image for booting with QEMU
(NOTE: use -j$(nproc) instead  of -j4 for parallel build)


modules and devicetree
● make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gn- modules
○ Build any kernel modules
● make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- dtbs
○ Build the devicetree




start qemu example:
qemu-system-aarch64 -m 256M -M virt -cpu cortex-a53 -nographic -smp 1 -kernel ${KERNEL_IMAGE} \
        -chardev stdio,id=char0,mux=on,logfile=${OUTDIR}/serial.log,signal=off \
        -serial chardev:char0 -mon chardev=char0\
        -append "rdinit=/bin/sh" -initrd ${INITRD_IMAGE}
