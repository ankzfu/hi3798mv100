# Hi3798mv100 (Huawei ec6108v9 IPTV) Linux compilation and burning blog
This article records the process of compiling kernel, burning uboot and flashing Ubuntu 16.04 rootfs for Huawei set-top box EC6108v9 (His Hi3798mv100 chip). At the same time, I made up for the relevant knowledge of uboot.
## Basic environment
Target board: IPTV retired Huawei set-top box EC6108v9 (hisilicon Hi3798mv100 2G 8G emmc)
Compilation environment: Ubuntu 16.04 32bit VM
HiSilicon linux kernel: HiSTBLinux for hi3798mv100 mv200
SDK: HiSTBLinuxV100R005C00SPC041B020

## Environment preparation

```
git clone https://github.com/glinuz/hi3798mv100
#Switch to working directory
cd HiSTBLinuxV100R005C00SPC041B020 #$SDK_path
#Install the required compilation tools, you can use the shell script with the SDK, or you can install it yourself
sh server_install.sh
#or
apt-get install gcc make gettext bison flex bc zlib1g-dev libncurses5-dev lzma
#Copy the predefined ones in the SDK
cp configs/hi3798mv100/hi3798mdmo1g_hi3798mv100_cfg.mak ./cfg.mak

source ./env.sh #SDK various environment variables
#Modify the compiled configuration as needed
make menuconfig
make build -j4 2>&1 | tee -a buildlog.txt

```
After the system is successfully created, you can find the compiled fastboot-burn.bin, bootargs.bin, and hi_kernel.bin in out/hi3798mv100, which are the uboot boot file, the uboot boot parameter configuration, and the linux kernel.
## Burn to eMMC using HiTool
See [hi3798mv100-ec6109.jpg] for the TTL connection diagram, and you can search the hitool tutorial for the specific programming scheme.

Hitool burn interface configuration build [hit00l-burn.png]

eMMC partitions are uboot 1M, bootargs 1M, kernel 8M, rootfs 128M, see [emmc_partitions.xml] for details.

If you modify the partition size and adjust the partition size, you need to modify the bootargs.txt and emmc_partitions.xml simultaneously.

configs/hi3798mv100/prebuilts/bootargs.txt, and regenerate the bootargs.bin file

git clone https://github.com/glinuz/hi3798mv100
#Switch to working directory
cd HiSTBLinuxV100R005C00SPC041B020 #$SDK_path
#Install the required compilation tools, you can use the shell script with the SDK, or you can install it yourself
sh server_install.sh
#or
apt-get install gcc make gettext bison flex bc zlib1g-dev libncurses5-dev lzma
#Copy the predefined ones in the SDK
cp configs/hi3798mv100/hi3798mdmo1g_hi3798mv100_cfg.mak ./cfg.mak

source ./env.sh #SDK various environment variables
#Modify the compiled configuration as needed
make menuconfig
make build -j4 2>&1 | tee -a buildlog.txt
````
After the system is successfully created, you can find the compiled fastboot-burn.bin, bootargs.bin, and hi_kernel.bin in out/hi3798mv100, which are the uboot boot file, the uboot boot parameter configuration, and the linux kernel.
## Burn to eMMC using HiTool
See [hi3798mv100-ec6109.jpg] for the TTL connection diagram, and you can search the hitool tutorial for the specific programming scheme.

Hitool burn interface configuration build [hit00l-burn.png]

eMMC partitions are uboot 1M, bootargs 1M, kernel 8M, rootfs 128M, see [emmc_partitions.xml] for details.

If you modify the partition size and adjust the partition size, you need to modify the bootargs.txt and emmc_partitions.xml simultaneously.

configs/hi3798mv100/prebuilts/bootargs.txt, and regenerate the bootargs.bin file

```
bootcmd=mmc read 0 0x1FFFFC0 0x1000 0x4000;bootm 0x1FFFFC0
bootargs=console=ttyAMA0,115200 root=/dev/mmcblk0p4 rootfstype=ext4 rootwait blkdevparts=mmcblk0:1M(fastboot),1M(bootargs),8M(kernel),128M(rootfs),-(system)

mkbootargs  -s 1M -r bootargs.txt  -o bootargs.bin
```
Bootcmd operation description: Starting from 2M bytes on the 0th mmc device block (0x1000 decimal 4096, 4096*512/1024=2M), read 16×512 bytes (0x4000 decimal 16384*512/1024= 8M) to memory 0x1FFFFC0 and boot from there.

Open the serial console for debugging. console=ttyAMA0,115200
The output of the uboot boot process is as follows:
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


Fastboot 3.3.0 (root@glinuz) (Jul 25 2020 - 08:25:47)

Fastboot:      Version 3.3.0
Build Date:    Jul 25 2020, 08:26:41
CPU:           Hi3798Mv100 
Boot Media:    eMMC
DDR Size:      1GB


MMC/SD controller initialization.
MMC/SD Card:
    MID:         0x15
    Read Block:  512 Bytes
    Write Block: 512 Bytes
    Chip Size:   7456M Bytes (High Capacity)
    Name:        "8GME4R"
    Chip Type:   MMC
    Version:     5.1
    Speed:       52000000Hz
    Mode:        DDR50
    Bus Width:   8bit
    Boot Addr:   0 Bytes
Net:   upWarning: failed to set MAC address


Boot Env on eMMC
    Env Offset:          0x00100000
    Env Size:            0x00010000
    Env Range:           0x00010000
ID_WORD have already been locked


SDK Version: HiSTBLinuxV100R005C00SPC041B020_20161028

Reserve Memory
    Start Addr:          0x3FFFE000
    Bound Addr:          0x8D24000
    Free  Addr:          0x3F8FC000
    Alloc Block:  Addr         Size
                  0x3FBFD000   0x400000
                  0x3F8FC000   0x300000

Press Ctrl+C to stop autoboot

MMC read: dev # 0, block # 4096, count 16384 ... 16384 blocks read: OK

84937034 Bytes/s
## Booting kernel from Legacy Image at 01ffffc0 ...
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
## Advanced compilation
### custom linux kernel
The ARM platform kernel configuration file adopts the defconfig format. The process of correctly using and saving deconfig is as follows:

source/kernel/linux-3.18.y/arch/arm/configs/hi3798mv100_defconfig
cd source/kernel/linux-3.18.y/
You can use hi3798mv100_defconfig-0812 provided by this git library
1. Backup hi3798mv100_defconfig first
2. make ARCH=arm hi3798mv100_defconfig #Generate standard linux kernel configuration .config file from defconfig
3. make ARCH=arm menuconfig #Modify the kernel configuration and save it
4. make ARCH=arm savedefconfig #Regenerate the defconfg file
5. cp defconfig arch/arm/configs/hi3798mv100_defconfig #Copy the defconfig file to the correct location.
6. make distclean #Clean up the files compiled and produced before
7. cd $SDK_path;make linux #Recompile the kernel

Kernel compilation parameters to pay attention to:

     Open devtmpfs, /dev filesystem

     open open by fhandle syscalls

     Turn on the cgroup function

### Modify uboot
```
source/boot/fastboot/include/configs godbox.h
#define CONFIG_SHOW_MEMORY_LAYOUT 1
#define CONFIG_SHOW_REG_INFO      1
#define CONFIG_SHOW_RESERVE_MEM_LAYOUT        1

or
cd $SDK_path;make hiboot CONFIG_SHOW_RESERVE_MEM_LAYOUT='y'
```
CONFIG_SHOW_RESERVE_MEM_LAYOUT='y' When compiling, turn on the output MEM information switch when uboot starts
## Modify uboot startup parameters at startup
During the uboot startup phase, Ctrl+C enters the uboot mode
```
 setenv bootargs console=tty1 console=ttyAMA0,115200 root=/dev/mmcblk0p4 rootfstype=ext4 rootwait blkdevparts=mmcblk0:1M(fastboot),1M(bootargs),8M(kernel),128M(rootfs),-(system)  ipaddr=192.168.10.100 gateway=192.168.10.1 netmask=255.255.255.0 netdev=eth0
 saveenv
 reset
 ```
## make ubuntu rootfs
```
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

LC_ALL=C LANGUAGE=C LANG=C chroot . /bin/bash  #以下命令在chroot环境bash执行
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
Make a rootfs image
````
make_ext4fs -l 128M -s rootfs_128M.ext4 ./rootfs
````
References

[1] https://wiki.ubuntu.com/ARM/RootfsFromScratch/QemuDebootstrap

[2] http://gnu-linux.org/building-ubuntu-rootfs-for-arm.html

## Flash package - binary file
file download release
fastboot-bin.bin uboot partition package
bootargs.bin uboot parameter partition package
hi_kernel.bin kernel partition package
rootfs_128m.ext root root partition package
emmc_partitions.xml flash partition configuration file
If you resize the partition, you need to regenerate bootargs.bin and adjust the partition configuration file.
Use Huawei hi-tool, emmc to burn

## uboot instructions
Many students asked about uboot startup. The main uboot parameters are as follows, emmc memory chip
```
bootcmd=mmc read 0 0x1FFFFC0 0x1000 0x4000;bootm 0x1FFFFC0
bootargs=console=ttyAMA0,115200 root=/dev/mmcblk0p4 rootfstype=ext4 rootwait blkdevparts=mmcblk0:1M(fastboot),1M(bootargs),8M(kernel),128M(rootfs),-(system)
```
bootcmd uboot boot boot: mmc read <device num> addr blk
Instruction Memory Address mmc Address Length
mmc read 0 0x1FFFFC0 0x1000 0x4000
bootm 0x1FFFFC0 #boot kernel from memory address
    
    
## other
Subsequently, packages such as python golang docker were added to debootstrap, and the rootfs was increased to 4GB, and the corresponding bootargs emmc_partition was modified.
