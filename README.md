# Hi3798mv100 (Huawei ec6108v9 IPTV) Linux Compilation and Flashing Blog

This article documents the process of compiling the kernel, flashing U-Boot, and installing the Ubuntu 16.04 root filesystem for the Huawei set-top box EC6108v9 (using the Hisilicon Hi3798mv100 chip). Additionally, I refreshed my knowledge of U-Boot.

## Basic Environment

Target Board: Retired Huawei EC6108v9 IPTV set-top box (Hisilicon Hi3798mv100 2GB RAM, 8GB eMMC)  
Compilation Environment: Ubuntu 16.04 32-bit VM  
Hisilicon Linux Kernel: HiSTBLinux for Hi3798mv100 mv200  
SDK: HiSTBLinuxV100R005C00SPC041B020  

## Environment Preparation

```bash
git clone https://github.com/glinuz/hi3798mv100
# Switch to the working directory
cd HiSTBLinuxV100R005C00SPC041B020  # $SDK_path
# Install the necessary compilation tools, either using the shell script from the SDK or installing them yourself
sh server_install.sh
# or
apt-get install gcc make gettext bison flex bc zlib1g-dev libncurses5-dev lzma
# Copy the pre-defined configurations from the SDK
cp configs/hi3798mv100/hi3798mdmo1g_hi3798mv100_cfg.mak ./cfg.mak

source ./env.sh  # SDK various environment variables
# Modify the compilation configurations as needed
make menuconfig
make build -j4 2>&1 | tee -a buildlog.txt
```
Once compilation is successful, you can find the compiled `fastboot-burn.bin`, `bootargs.bin`, and `hi_kernel.bin` in `out/hi3798mv100`, which are the U-Boot boot files, U-Boot boot parameter configurations, and the Linux kernel, respectively.

## Using HiTool to Flash to eMMC

Refer to the TTL connection diagram [hi3798mv100-ec6109.jpg], and you can search for HiTool tutorials for the specific flashing procedure.

The HiTool flashing interface configuration can be found in [hit00l-burn.png].

The eMMC partitions are `uboot 1M`, `bootargs 1M`, `kernel 8M`, and `rootfs 128M`. For specifics, see [emmc_partitions.xml].

If the partition sizes are modified, adjust them and synchronize changes in `bootargs.txt` and `emmc_partitions.xml`.

Edit `configs/hi3798mv100/prebuilts/bootargs.txt` and regenerate the `bootargs.bin` file:

```bash
bootcmd=mmc read 0 0x1FFFFC0 0x1000 0x4000;bootm 0x1FFFFC0
bootargs=console=ttyAMA0,115200 root=/dev/mmcblk0p4 rootfstype=ext4 rootwait blkdevparts=mmcblk0:1M(fastboot),1M(bootargs),8M(kernel),128M(rootfs),-(system)

mkbootargs -s 1M -r bootargs.txt -o bootargs.bin
```
The `bootcmd` operation description: Start reading from the first MMC device block, 2MB (0x1000 in decimal is 4096; 4096 * 512 / 1024 = 2M), reading 16×512 bytes (0x4000 in decimal is 16384; 16384 * 512 / 1024 = 8M) into memory at 0x1FFFFC0, and boot from there.

Open the serial console for debugging: `console=ttyAMA0,115200`.  
The U-Boot startup process output is as follows:

```
Bootrom start
Boot from eMMC
Starting fastboot ...

System startup
DDRS
Reg Version:  v1.1.0
Reg Time:     2016/1/18  14:01:18
Reg Name:     hi3798mdmo1g_hi3798mv100_ddr3_1gbyte_16bitx2_4layers_emmc.reg

Jump to DDR

...

Fastboot 3.3.0 (root@glinuz) (Jul 25 2020 - 08:25:47)

Fastboot:      Version 3.3.0
Build Date:    Jul 25 2020, 08:26:41
CPU:           Hi3798Mv100 
Boot Media:    eMMC
DDR Size:      1GB

...

Booting kernel from Legacy Image at 01ffffc0 ...
   Image Name:   Linux-3.18.24_s40
   Image Type:   ARM Linux Kernel Image (uncompressed)
   Data Size:    6959232 Bytes = 6.6 MiB
   Load Address: 02000000
   Entry Point:  02000000
   Verifying Checksum ... OK
   XIP Kernel Image ... OK
OK
ATAGS [0x00000100 - 0x00000300], 512Bytes

Starting kernel ...
```
## Advanced Compilation

### Custom Linux Kernel

For the ARM platform, the kernel configuration file uses the defconfig format. The correct usage and saving process is as follows:

```bash
source/kernel/linux-3.18.y/arch/arm/configs/hi3798mv100_defconfig 
cd source/kernel/linux-3.18.y/
```
You can use the `hi3798mv100_defconfig-0812` provided by this Git repository.

1. First, backup `hi3798mv100_defconfig`
2. `make ARCH=arm hi3798mv100_defconfig` # Generate the standard Linux kernel configuration `.config` file from defconfig.
3. `make ARCH=arm menuconfig` # Modify the kernel configuration and save.
4. `make ARCH=arm savedefconfig` # Regenerate the defconfig file.
5. `cp defconfig arch/arm/configs/hi3798mv100_defconfig`  # Copy the defconfig file to the correct location.
6. `make distclean` # Clean up previously compiled files.
7. `cd $SDK_path;make linux`  # Recompile the kernel.

Key kernel compilation parameters to pay attention to:

- Enable devtmpfs, the `/dev` filesystem
- Enable open by fhandle syscalls
- Enable cgroup functionality

### Modify U-Boot

```c
source/boot/fastboot/include/configs godbox.h
#define CONFIG_SHOW_MEMORY_LAYOUT 1
#define CONFIG_SHOW_REG_INFO      1
#define CONFIG_SHOW_RESERVE_MEM_LAYOUT        1

or
cd $SDK_path;make hiboot CONFIG_SHOW_RESERVE_MEM_LAYOUT='y'
```
When `CONFIG_SHOW_RESERVE_MEM_LAYOUT='y'` is set during compilation, it enables the output of memory information during U-Boot startup.

## Modify U-Boot Boot Parameters at Startup

During the U-Boot startup stage, press Ctrl+C to enter the U-Boot mode:

```bash
setenv bootargs console=tty1 console=ttyAMA0,115200 root=/dev/mmcblk0p4 rootfstype=ext4 rootwait blkdevparts=mmcblk0:1M(fastboot),1M(bootargs),8M(kernel),128M(rootfs),-(system)  ipaddr=192.168.10.100 gateway=192.168.10.1 netmask=255.255.255.0 netdev=eth0
saveenv
reset
```

## Create Ubuntu Root Filesystem

```bash
apt-get install binfmt-support debootstrap qemu qemu-user-static
cd;mkdir rootfs
debootstrap --arch=armhf --variant=minbase  --foreign --include=locales,util-linux,apt-utils,ifupdown,systemd-sysv,iproute2,curl,wget,expect,ca-certificates,openssh-server,isc-dhcp-client,vim-tiny,bzip2,cpio,usbutils,netbase,parted,jq,bc,crda,wireless-tools,iw stretch rootfs http://mirrors.ustc.edu.cn/debian/

cd rootfs
cp /usr/bin/qemu-arm-static usr/bin
mount -v --bind /dev dev
mount -vt devpts devpts dev/pts -o gid=5,mode=620
mount -t proc proc proc
mount -t sysfs sysfs sys
mount -t tmpfs tmpfs run
LC_ALL=C LANGUAGE=C LANG=C chroot . /debootstrap/debootstrap --second-stage
LC_ALL=C LANGUAGE=C LANG=C chroot . dpkg --configure -a

LC_ALL=C LANGUAGE=C LANG=C chroot . /bin/bash  # The following commands are executed in a chroot environment bash
mkdir /proc
mkdir /tmp
mkdir /sys
mkdir /root

mknod /dev/console c 5 1
mknod /dev/ttyAMA0 c 204 64
mknod /dev/ttyAMA1 c 204 65

mknod /dev/ttyS000 c 204 64
mknod /dev/null    c 1   3
mknod /dev/urandom   c 1   9
mknod /dev/zero    c 1   5
mknod /dev/random    c 1   8
mknod /dev/tty    c 5   0

echo "nameserver 114.114.114.114" > /etc/resolv.conf
echo "hi3798m" > /etc/hostname
echo "Asia/Shanghai" > /etc/timezone
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

echo "en_US.UTF-8 UTF-8" > etc/locale.gen
echo "zh_CN.UTF-8 UTF-8" >> etc/locale.gen
echo "zh_CN.GB2312 GB2312" >> etc/locale.gen
echo "zh_CN.GBK GBK" >> etc/locale.gen

locale-gen

echo "LANG=en_US.UTF-8" > /etc/locale.conf

echo "deb http://mirrors.ustc.edu.cn/debian/  stretch main contrib non-free" >  /etc/apt/sources.list

ln -s /lib/systemd/system/getty@.service /etc/systemd/system/getty.target.wants/getty@ttyAMA0.service
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config

apt autoremove
apt-get autoclean
apt-get clean
apt clean
```

Creating the root filesystem image:

```bash
make_ext4fs -l 128M -s rootfs_128M.ext4 ./rootfs
```

## Flashing Package - Binary Files
Download the file release:

- `fastboot-bin.bin` - U-Boot partition package
- `bootargs.bin` - U-Boot parameter partition package
- `hi_kernel.bin` - Kernel partition package
- `rootfs_128m.ext` - Root filesystem package
- `emmc_partitions.xml` - Flashing partition configuration file

If you adjust the partition sizes, you need to regenerate `bootargs.bin` and adjust the partition configuration file. Use Huawei Hi-Tool for eMMC flashing.

## U-Boot Explanation

Many students ask about U-Boot startup; key U-Boot parameters regarding the eMMC storage chip are as follows:

```
bootcmd=mmc read 0 0x1FFFFC0 0x1000 0x4000;bootm 0x1FFFFC0
bootargs=console=ttyAMA0,115200 root=/dev/mmcblk0p4 rootfstype=ext4 rootwait blkdevparts=mmcblk0:1M(fastboot),1M(bootargs),8M(kernel),128M(rootfs),-(system)
```
The `bootcmd` U-Boot boot command: `mmc read <device num> addr blk` indicates the memory address and the internal MMC address and length.

## Others

Subsequently, I added Python, Go, Docker, and other software packages in `debootstrap`, enlarged the root filesystem to 4GB, and modified the corresponding `bootargs` and `emmc_partition`.
