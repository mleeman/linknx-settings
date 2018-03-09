# EIB/KNX on SheevaPlug (using GNU/Debian 6.0)

EIB/KNX on Sheeva Plug using a clean GNU/Debian Squeeze system on the
NAND flash. This document was generated with asciidoc. The source file
can be downloaded in case you want to generated other formats.
## Cleaning the house; installing GNU/Debian Squeeze on the NAND flash
### Introduction

The Sheeva Plug is an excellent platform to do all kinds of
applications. Combined with its' small form factor and low power
consumption for a very decent performance, it is a perfect tool for
running control applications on.

EIB/KNX on Sheeva Plug describes how to install GNU/Debian 5.0 on
a SheevaPlug, this document describes the same, but the more recent
GNU/Debian 6.0 aka Squeeze. In the remainder of the text, there are
two options:

1. Creating a system from scratch; this includes creating a cross
   compiler, building the kernel and creating the base root filesystem.
2. Using the provided images to flash the SheevaPlug.

Though the former option is interesting for developers; the latter option
provides a low threshold for people that do not care about the inner
workings of creating a foreign architecture root filesystem and just
care about getting the system up and running with a recent distribution.

### Starting up: Creating a cross compiler
In this section, the cross compiler will be created. You can skip this
if you just want to flash your plug with Squeeze.

See this section for simple flashing of your system.

#### Files

#### Buildroot configuration file

Before the work can be started, a cross compiler needs to be created:
a compiler that will create binary code for the ARM 926 CPU on the
SheevaPlug, but will run on the development machine (typically an 64
bit x86 machine). Buildroot is the tool of choice. Download any release
(this HOWTO is done with the 2011.11); and extract it on the development
machine.

Buildroot had little dependencies itself; and is configured with
a simple .config file. This document opted for a generic toolchain
using glibc instead of the normal uclibc that is typically associated
with buildroot. This will enable to re-use the toolchain in our final
GNU/Debian system. Explore the options, or download the config file,
extract it and place it in the extracted buildroot source directory.

Type make, sit back and enjoy a couple cups of coffee. The build process
will finish with a toolchain and a couple of targets, including some
typical embedded root filesystem targets that we can ignore in our
setup. The only real target we’re interested in, is the u-boot.kwb
file in the output/images/ directory.

### Starting up: Creating the base root filesystem: Part 1

In this section, the cross compiler will be created. You can skip this
if you just want to flash your plug with Squeeze.

See this section for simple flashing of your system.

#### Files

1. Debian Staging

debootstrap is used to create a foreign root filesystem for the armel
architecture. Note that this is the GNU/Debian architecture name (armel)
and not the kernel one (arm).

    $ mkdir sheeva-armel-squeeze/
    $ sudo cdebootstrap --arch armel --foreign squeeze sheeva-armel-squeeze/ \
        ftp://ftp.nl.debian.org/debian

Before continuing, some minor modifications are done.
Create an image of the root filesystem that will be used to finish the
installation:

    $ tar cf /home/marc/sheevaplug-images/sheeva-armel-squeeze-step1.tar -C sheeva-armel-squeeze/ .
    $ xz /home/marc/sheevaplug-images/sheeva-armel-squeeze-step1.tar

### Starting up: Compiling the Linux kernel: Part 1
In this section, the cross compiler will be created. You can skip this
if you just want to flash your plug with Squeeze.

See this section for simple flashing of your system.

#### Files

1. config file

In the first phase, the initial base root filesystem install needs
to be finished. Since the final goal is to have a clean Squeeze image
that allows simple flashing onto the internal NAND flash, this is an
intermediary step. As such, a simple USB stick is used to store the
filesystem on and the current kernel needs to support booting from USB
with an ext2 filesystem.

Even though buildroot can perfectly compile the kernel, this is done
out of tree, using a separate copy of the kernel. Download the Linux
kernel from kernel.org and extract it. In the top source directory,
place the extracted config file as .config.

  $ make ARCH=arm CROSS_COMPILE=arm-linux- uImage

### Starting up: Creating the base root filesystem: Part 2
In this section, the cross compiler will be created. You can skip this
if you just want to flash your plug with Squeeze.

See this section for simple flashing of your system.

#### Files

1. u-boot.kwb
2. uImage

Use a VFAT formatted memory stick and copy the u-boot.kwb file (built
with buildroot) and the uImage onto that. Connect the SheevaPlug to
your machine with the USB cable and power it up. Make certain you are
ready to interrupt the boot cycle with your favourite serial connector
(e.g. minicom, screen). The serial settings are <u>115200 8N1</u> on
device node /dev/ttyUSB0.

  nand device 0
  usb start

Erase the bootloader location and burn the newly compiled bootloader
onto the NAND flash.

    nand erase 0x0 0xa0000
    fatload usb 0:1 0x8000000 /u-boot.kwb
    nand write.e 0x8000000 0x0 0xa0000
    reset

At this point, the system should boot with the newly compiled
bootloader. Make certain to interrupt the boot cycle before the counter
hits 0.

resetting ...

U-Boot 2011.06 (Dec 01 2011 - 13:37:18)
Marvell-Sheevaplug

SoC:   Kirkwood 88F6281_A0
DRAM:  512 MiB
NAND:  512 MiB
In:    serial
Out:   serial
Err:   serial
Net:   egiga0
88E1116 Initialized on egiga0
Hit any key to stop autoboot:  0
Marvell>>

In a next step, the existing kernel is removed and replaced by our USB
capable kernel.

    nand erase 0x100000 0x400000
    fatload usb 0:1 0x8000000 /uImage
    nand write.e 0x8000000 0x100000 0x400000

In order to boot from the USB stick, the behaviour of the bootloader is
adjusted. The following commands will erase settings and modify them
to boot from the USB stick. Finally, the settings are saved (verify
with printenv).

    setenv bootargs
    setenv boot_nand
    setenv bootargs_console 'console=ttyS0,115200'
    setenv bootargs_root 'root=/dev/sda1 rootdelay=10'
    setenv bootcmd_usb 'usb start; nand read.e 0x2000000 0x100000 0x400000'
    setenv bootcmd 'setenv bootargs $(bootargs_console) $(bootargs_root); \
        run bootcmd_usb; bootm 0x2000000'
    saveenv
    reset

The system reboots, the kernel starts and mounts the USB based
filesystem. Instead of offering the user a shell; the installation of
GNU/Debian Squeeze is continued: packages will be installed (already
available on the image). If your installation stops for some reason,
there is some housekeeping to do, if all goes fine; a brandy new Squeeze
system will be running of your USB stick. In my case; the installation
failed with a kernel panic; so I had to clean up, start the system and
you should be dropped into a shell on the serial console. If not; a lot
of the modifications can be done on any other Linux system by mounting
the stick as ext2:

    # mv /sbin/init.REAL to /sbin/init

Make certain that init is spawning a serial console. there are two
options; one with a login (first line) or spawn a bash shell in any case
(second line).A good option is to use a bash initially; but replace it
with a login once the system is properly configured.

    # T0:23:respawn:/sbin/getty -L ttyS0 115200 linux
    T0:23:respawn:/bin/bash

There are a number of things that need further taking care of like:
- Set the root password (passwd). - Allow rc policy changes (rm
/usr/sbin/policy-rc.d)

Finally; add a proper network configuration in /etc/network/interfaces
and add a correct /etc/fstab:

# cat /etc/fstab
# /etc/fstab: static file system information.
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
tmpfs /lib/init/rw  tmpfs rw,nosuid,mode=0755 0 0
proc  /proc proc  rw,noexec,nosuid,nodev 0 0
sysfs /sys  sysfs rw,noexec,nosuid,nodev 0 0
varrun  /var/run  tmpfs rw,nosuid,mode=0755 0 0
varlock /var/lock tmpfs rw,noexec,nosuid,nodev,mode=1777 0 0
udev  /dev  tmpfs rw,mode=0755 0 0
tmpfs /dev/shm  tmpfs rw,nosuid,nodev 0 0
devpts  /dev/pts  devpts rw,noexec,nosuid,gid=5,mode=620 0 0
/dev/root / rootfs  rw 0 0
tmpfs /var/cache/apt  tmpfs defaults,noatime

and for the network configuration, default back to DHCP:

    # cat /etc/network/interfaces

    # The loopback network interface
    auto lo
    iface lo inet loopback

    # The primary network interface
    allow-hotplug eth0
    iface eth0 inet dhcp

Currently, only the root user is available on the system. Create a default
user to use e.g. knx/knx. Note that this user will also be created when
installing the knx packages.

    # adduser knx

Since part of the filesystem is mounted on memory (see /etc/fstab);
some crucial directories are not created by default. These directories
can be created at boot time. Add the following lines to /etc/rc.local.

    mkdir -p /var/log/knx/
    mkdir -p /var/run/knx/
    chown knx.nogroup /var/log/knx/
    chown knx.nogroup /var/run/knx/
    mkdir -p /var/cache/apt/archives/partial

Before finishing up; some extra packages are installed:

    # install openssh-server ntp ntpdate vim
    # apt-get clean

Shut down the system. At this point, the clean root filesystem is
available on the USB memory stick.

### Starting up: Compiling the Linux kernel: Part 2

In this section, the cross compiler will be created. You can skip this
if you just want to flash your plug with Squeeze.

See this section for simple flashing of your system.

Files

1. ARMEL 3.1.4 modules
2. uImage

Since the kernel that was compiled previously was optimised for booting
from an USB memory stick; a final kernel needs to be created to boot
from the internal NAND flash, using UbiFS instead of ext2. For that,
another configuration is used.

    $ make ARCH=arm CROSS_COMPILE=arm-linux- uImage
    $ make ARCH=arm CROSS_COMPILE=arm-linux- modules
    $ sudo make INSTALL_MOD_PATH=/home/marc/sheevaplug-build/modules-3.1.4/ \
        ARCH=arm CROSS_COMPILE=arm-linux- modules_install

Create an archive of the modules.

$ sudo tar cf /home/marc/sheevaplug-images/sheevaplug-modules-3.1.4.tar \
    -C /home/marc/sheevaplug-build/modules-3.1.4/ .
$ xz /home/marc/sheevaplug-images/sheevaplug-modules-3.1.4.tar

At this point; all is available to create the final images.

### Starting up: Creating the base root filesystem: Part 3
In this section, the cross compiler will be created. You can skip this
if you just want to flash your plug with Squeeze.

See this section for simple flashing of your system.

Files

1. SheevaPlug target
2. SheevaPlug UBI target

After creating a filesystem and compiling the kernel with modules; the
both need to be combined. Use the USB stick, mount it and extract the
modules into the file system.

    $ cd /media/usb1
    $ sudo tar xf /home/marc/sheevaplug-images/sheevaplug-modules-3.1.4.tar.xz
    $ sudo find . | while read i ; do sudo touch $i; done

The filesystem on the memory stick is now a perfect image of what needs
to be burnt into flash. Before continuing, create a snapshot. The same
technique is used as for creating an image of the modules.

First, an image is created of the filesystem in the USB stick (backup
purposes).

    $ sudo tar cf /home/marc/sheevaplug-images/sheeva-armel-squeeze-target.tar \
        -C /media/usb1/ .

Finally, create the filesystem image to flash. The utilities are a part
of mtd-utils.

[mleeman@bane debian-rootfs]$ sudo mkfs.ubifs -r sheeva-rootfs -m 2048 \
    -e 129024 -c 4096 -o ubifs.img -x zlib
[mleeman@bane debian-rootfs]$ cat ubi.cfg
[ubifs]
mode=ubi
image=ubifs.img
vol_id=0
vol_size=256MiB
vol_type=dynamic
vol_name=rootfs
vol_flags=autoresize
[mleeman@bane debian-rootfs]$ sudo ubinize -o ubi.img -m 2048 \
    -p 128KiB -s 512 ubi.cfg

### Install GNU/Debian 6.0 on the internal NAND flash

Files

1. u-boot.kwb
2. uImage
3. SheevaPlug UBI target

At this point, the final images are created. There are several ways of
getting the images onto the SheevaPlug, but having a simple USB stick
is probably the simplest (alternatives can be e.g. TFTP).

Place the 3 images on an VFAT formatted USB stick:

    u-boot.kwb
    uImage
    ubi.img

Initialise the attached USB drive. Note that I did this with a USB
disk that contains a partition, I don’t know if the bootloader will
recognise one that does not have a partition

    nand device 0
    usb start
    setenv quiet 0

write u-boot. This is not really required, but heavily suggested to
have better USB support wrt the standard firmware. After writing the
bootloader, press reset and let it use the new bootloader.

    nand erase 0x0 0xa0000
    fatload usb 0:1 0x8000000 /u-boot.kwb
    nand write.e 0x8000000 0x0 0xa0000
    reset

write the kernel to the NAND flash

    nand device 0
    usb start
    setenv quiet 0
    nand erase 0x100000 0x400000
    fatload usb 0:1 0x8000000 /uImage
    nand write.e 0x8000000 0x100000 0x400000

Finally, write the UbiFS root filesystem:

    nand erase 0x500000 0x1fb00000
    fatload usb 0:1 0x8000000 /ubi.img
    nand write.e 0x8000000 0x500000 0x1fb00000

The write size is the full flash; if there are too many bad blocks;
just adjust the write size to a bit smaller (e.g. 0x1f000000). Don’t
worry too much about the bad blocks; bad blocks is pretty normal for
low cost NAND flash (in comparison with more expensive NOR flash).

### Booting in the final system

Wrapping up, we signal our bootloader to start our system by default,
we also need to set that we’re booting a mainline kernel.

    setenv bootargs 'console=ttyS0,115200 ubi.mtd=2 root=ubi0:rootfs rootfstype=ubifs'
    setenv boot_nand 'nand read.e 0x2000000 0x100000 0x400000'
    setenv bootcmd 'run boot_nand; bootm 0x2000000'
    setenv ethaddr f0:ad:4e:00:36:62
    saveenv
    reset

Do not forget to set the MAC address to the value that is printed on
the plug itself, otherwise, the bootloader will load a default one.

Remove the memory stick (not required), and reboot. If all works out,
the following output should appear:

Marvell>>   reset
resetting ...

U-Boot 2011.06 (Dec 01 2011 - 13:37:18)
Marvell-Sheevaplug

SoC:   Kirkwood 88F6281_A1
DRAM:  512 MiB
NAND:  512 MiB
In:    serial
Out:   serial
Err:   serial
Net:   egiga0
88E1116 Initialized on egiga0
Hit any key to stop autoboot:  0

NAND read: device 0 offset 0x100000, size 0x400000
 4194304 bytes read: OK
## Booting kernel from Legacy Image at 02000000 ...
   Image Name:   Linux-3.1.4
   Image Type:   ARM Linux Kernel Image (uncompressed)
   Data Size:    1981064 Bytes = 1.9 MiB
   Load Address: 00008000
   Entry Point:  00008000
   Verifying Checksum ... OK
   Loading Kernel Image ... OK
OK

Starting kernel ...

Uncompressing Linux... done, booting the kernel.
[    0.000000] Initializing cgroup subsys cpuset
[    0.000000] Initializing cgroup subsys cpu
[    0.000000] Linux version 3.1.4 (mleeman@cypher) (gcc version 4.4.3 (crosstool-NG 1.13.2 - buildroot 2011.11) ) #7 Fri Dec 2 14:05:16 CET 2011
[    0.000000] CPU: Feroceon 88FR131 [56251311] revision 1 (ARMv5TE), cr=00053977
[    0.000000] CPU: VIVT data cache, VIVT instruction cache
[    0.000000] Machine: Marvell SheevaPlug Reference Board
[    0.000000] Memory policy: ECC disabled, Data cache writeback
[    0.000000] Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 130048
[    0.000000] Kernel command line: console=ttyS0,115200 ubi.mtd=2 root=ubi0:rootfs rootfstype=ubifs
[    0.000000] PID hash table entries: 2048 (order: 1, 8192 bytes)
[    0.000000] Dentry cache hash table entries: 65536 (order: 6, 262144 bytes)
[    0.000000] Inode-cache hash table entries: 32768 (order: 5, 131072 bytes)
[    0.000000] Memory: 512MB = 512MB total
[    0.000000] Memory: 515732k/515732k available, 8556k reserved, 0K highmem
[    0.000000] Virtual kernel memory layout:
[    0.000000]     vector  : 0xffff0000 - 0xffff1000   (   4 kB)
[    0.000000]     fixmap  : 0xfff00000 - 0xfffe0000   ( 896 kB)
[    0.000000]     DMA     : 0xffc00000 - 0xffe00000   (   2 MB)
[    0.000000]     vmalloc : 0xe0800000 - 0xfe800000   ( 480 MB)
[    0.000000]     lowmem  : 0xc0000000 - 0xe0000000   ( 512 MB)
[    0.000000]     modules : 0xbf000000 - 0xc0000000   (  16 MB)
[    0.000000]       .text : 0xc0008000 - 0xc0375ff8   (3512 kB)
[    0.000000]       .init : 0xc0376000 - 0xc0398000   ( 136 kB)
[    0.000000]       .data : 0xc0398000 - 0xc03bd8e0   ( 151 kB)
[    0.000000]        .bss : 0xc03bd904 - 0xc03edc78   ( 193 kB)
[    0.000000] NR_IRQS:114
[    0.000000] sched_clock: 32 bits at 200MHz, resolution 5ns, wraps every 21474ms
[    0.000000] Console: colour dummy device 80x30
[   15.331166] Calibrating delay loop... 1191.11 BogoMIPS (lpj=5955584)
[   15.421067] pid_max: default: 32768 minimum: 301
[   15.421173] Security Framework initialized
[   15.421190] SELinux:  Disabled at boot.
[   15.421240] Mount-cache hash table entries: 512
[   15.421492] Initializing cgroup subsys cpuacct
[   15.421511] Initializing cgroup subsys devices
[   15.421520] Initializing cgroup subsys freezer
[   15.421529] Initializing cgroup subsys net_cls
[   15.421582] CPU: Testing write buffer coherency: ok
[   15.423602] print_constraints: dummy:
[   15.423863] NET: Registered protocol family 16
[   15.425179] Kirkwood: MV88F6281-A1, TCLK=200000000.
[   15.425192] Feroceon L2: Cache support initialised.
[   15.433773] bio: create slab <bio-0> at 0
[   15.434331] vgaarb: loaded
[   15.435659] Switching to clocksource orion_clocksource
[   15.441055] Switched to NOHz mode on CPU #0
[   15.447674] NET: Registered protocol family 2
[   15.447866] IP route cache hash table entries: 4096 (order: 2, 16384 bytes)
[   15.448470] TCP established hash table entries: 16384 (order: 5, 131072 bytes)
[   15.448824] TCP bind hash table entries: 16384 (order: 4, 65536 bytes)
[   15.449004] TCP: Hash tables configured (established 16384 bind 16384)
[   15.449014] TCP reno registered
[   15.449024] UDP hash table entries: 256 (order: 0, 4096 bytes)
[   15.449046] UDP-Lite hash table entries: 256 (order: 0, 4096 bytes)
[   15.449247] NET: Registered protocol family 1
[   15.449412] NetWinder Floating Point Emulator V0.97 (double precision)
[   15.450273] audit: initializing netlink socket (disabled)
[   15.450308] type=2000 audit(0.100:1): initialized
[   15.461224] VFS: Disk quotas dquot_6.5.2
[   15.461313] Dquot-cache hash table entries: 1024 (order 0, 4096 bytes)
[   15.461426] JFFS2 version 2.2. (NAND) (SUMMARY)  ?? 2001-2006 Red Hat, Inc.
[   15.461734] msgmni has been set to 1007
[   15.462423] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 253)
[   15.462437] io scheduler noop registered
[   15.462445] io scheduler deadline registered
[   15.462491] io scheduler cfq registered (default)
[   15.462544] mv_xor_shared mv_xor_shared.0: Marvell shared XOR driver
[   15.462571] mv_xor_shared mv_xor_shared.1: Marvell shared XOR driver
[   15.495724] mv_xor mv_xor.0: Marvell XOR: ( xor cpy )
[   15.535722] mv_xor mv_xor.1: Marvell XOR: ( xor fill cpy )
[   15.575720] mv_xor mv_xor.2: Marvell XOR: ( xor cpy )
[   15.615720] mv_xor mv_xor.3: Marvell XOR: ( xor fill cpy )
[   15.616261] Serial: 8250/16550 driver, 2 ports, IRQ sharing disabled
[   15.636953] serial8250.0: ttyS0 at MMIO 0xf1012000 (irq = 33) is a 16550A
[   16.042249] console [ttyS0] enabled
[   16.052905] brd: module loaded
[   16.057241] NAND device: Manufacturer ID: 0xec, Chip ID: 0xdc (Samsung NAND 512MiB 3,3V 8-bit)
[   16.065924] Scanning device for bad blocks
[   16.374282] Creating 3 MTD partitions on "orion_nand":
[   16.379458] 0x000000000000-0x000000100000 : "u-boot"
[   16.385577] ftl_cs: FTL header not found.
[   16.390252] 0x000000100000-0x000000500000 : "uImage"
[   16.396185] ftl_cs: FTL header not found.
[   16.400802] 0x000000500000-0x000020000000 : "root"
[   16.407041] ftl_cs: FTL header not found.
[   16.412023] UBI: attaching mtd2 to ubi0
[   16.415898] UBI: physical eraseblock size:   131072 bytes (128 KiB)
[   16.422194] UBI: logical eraseblock size:    129024 bytes
[   16.427627] UBI: smallest flash I/O unit:    2048
[   16.432353] UBI: sub-page size:              512
[   16.436991] UBI: VID header offset:          512 (aligned 512)
[   16.442852] UBI: data offset:                2048
[   17.110985] UBI: max. sequence number:       0
[   17.150386] UBI: volume 0 ("rootfs") re-sized from 2081 to 4012 LEBs
[   17.157353] UBI: attached mtd2 to ubi0
[   17.161119] UBI: MTD device name:            "root"
[   17.166041] UBI: MTD device size:            507 MiB
[   17.171024] UBI: number of good PEBs:        4056
[   17.175751] UBI: number of bad PEBs:         0
[   17.180215] UBI: number of corrupted PEBs:   0
[   17.184671] UBI: max. allowed volumes:       128
[   17.189309] UBI: wear-leveling threshold:    4096
[   17.194028] UBI: number of internal volumes: 1
[   17.198491] UBI: number of user volumes:     1
[   17.202948] UBI: available PEBs:             0
[   17.207411] UBI: total number of reserved PEBs: 4056
[   17.212392] UBI: number of PEBs reserved for bad PEB handling: 40
[   17.218521] UBI: max/mean erase counter: 1/0
[   17.222803] UBI: image sequence number:  0
[   17.226950] UBI: background thread "ubi_bgt0d" started, PID 282
[   17.236044] mv643xx_eth: MV-643xx 10/100/1000 ethernet driver version 1.4
[   17.255710] mv643xx_eth smi: probed
[   17.286129] mv643xx_eth_port mv643xx_eth_port.0: eth0: port 0 with MAC address 02:50:43:44:71:d3
[   17.306067] mousedev: PS/2 mouse device common for all mice
[   17.325791] rtc-mv rtc-mv: rtc core: registered rtc-mv as rtc0
[   17.331893] i2c /dev entries driver
[   17.346124] cpuidle: using governor ladder
[   17.365981] cpuidle: using governor menu
[   17.376781] TCP cubic registered
[   17.380028] NET: Registered protocol family 17
[   17.384501] Registering the dns_resolver key type
[   17.406360] registered taskstats version 1
[   17.410939] rtc-mv rtc-mv: setting system clock to 2011-05-11 02:35:05 UTC (1305081305)
[   17.646637] UBIFS: mounted UBI device 0, volume 0, name "rootfs"
[   17.652677] UBIFS: file system size:   516225024 bytes (504126 KiB, 492 MiB, 4001 LEBs)
[   17.660734] UBIFS: journal size:       9033728 bytes (8822 KiB, 8 MiB, 71 LEBs)
[   17.668084] UBIFS: media format:       w4/r0 (latest is w4/r0)
[   17.673942] UBIFS: default compressor: zlib
[   17.678155] UBIFS: reserved for root:  0 bytes (0 KiB)
[   17.684178] VFS: Mounted root (ubifs filesystem) on device 0:12.
[   17.690635] Freeing init memory: 136K
INIT: version 2.88 booting
Using makefile-style concurrent boot in runlevel S.
Starting the hotplug events dispatcher: udevd.
Synthesizing the initial hotplug events...done.
[   19.718855] usbcore: registered new interface driver usbfs
[   19.724466] usbcore: registered new interface driver hub
Waiting for /dev to be fully populated...[   19.749679] mmc0: mvsdio driver initialized, lacking card detect (fall back to polling)
[   19.822679] usbcore: registered new device driver usb
[   19.847656] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[   19.854264] orion-ehci orion-ehci.0: Marvell Orion EHCI
[   19.859592] orion-ehci orion-ehci.0: new USB bus registered, assigned bus number 1
[   19.895774] orion-ehci orion-ehci.0: irq 19, io mem 0xf1050000
[   19.916295] orion-ehci orion-ehci.0: USB 2.0 started, EHCI 1.00
[   19.922324] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002
[   19.929169] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[   19.936432] usb usb1: Product: Marvell Orion EHCI
[   19.941161] usb usb1: Manufacturer: Linux 3.1.4 ehci_hcd
[   19.946501] usb usb1: SerialNumber: orion-ehci.0
[   19.951485] hub 1-0:1.0: USB hub found
[   19.955258] hub 1-0:1.0: 1 port detected
done.
[   20.275763] usb 1-1: new high speed USB device number 2 using orion-ehci
Activating swap...done.
[   20.447066] usb 1-1: New USB device found, idVendor=0204, idProduct=6025
[   20.453807] usb 1-1: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[   20.460997] usb 1-1: Product: Flash Disk
[   20.465463] usb 1-1: Manufacturer: CBM
[   20.469682] usb 1-1: SerialNumber: 0319110061A43702
[   20.558760] SCSI subsystem initialized
[   20.584874] usbcore: registered new interface driver uas
[   20.612098] Initializing USB Mass Storage driver...
[   20.635864] scsi0 : usb-storage 1-1:1.0
[   20.640384] usbcore: registered new interface driver usb-storage
[   20.646442] USB Mass Storage support registered.
Cleaning up ifupdown....
Setting up networking....
Loading kernel modules...done.
Activating lvm and md swap...done.
Checking file systems...fsck from util-linux-ng 2.17.2
done.
Mounting local filesystems...done.
Activating swapfile swap...done.
Cleaning up temporary files....
Configuring network interfaces...done.
Cleaning up temporary files....
Setting kernel variables ...done.
INIT: Entering runlevel: 2
Using makefile-style concurrent boot in runlevel 2.
Starting enhanced syslogd: rsyslogd.
Starting periodic command scheduler: cron.
Starting NTP server: ntpd.
[   24.693650] NET: Registered protocol family 10
[   24.700348] ADDRCONF(NETDEV_UP): eth0: link is not ready
Starting OpenBSD Secure Shell server: sshd[   25.149424] sshd (750): /proc/750/oom_adj is deprecated, please use /proc/750/oom_score_adj instead.
.
[   25.712055] mv643xx_eth_port mv643xx_eth_port.0: eth0: link up, 100 Mb/s, full duplex, flow control disabled
[   25.722729] ADDRCONF(NETDEV_CHANGE): eth0: link becomes ready

Debian GNU/Linux 6.0 knx ttyS0

knx login:

The password for the root user is root, change it and re-generated the ssh keys

rm /etc/ssh/ssh_host*
ssh-keygen -t dsa -f /etc/ssh/ssh_host_dsa_key -N ``
ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key -N ``

The system is now prepared with a GNU/Debian Lenny system to install
the remainder of the software on.

2. Customising the system to run KNX/EIB
2.1. Preparing our KNX/EIB System

    eibd
    linknx: linknx
    webknx: webknx

First, add the scorpius repository to your sources.list

    deb http://scorpius.homelinux.org/~marc/debian squeeze main contrib non-free

Install lighttpd in order to get a webserver (boa or apache will do as well)

    $ sudo apt-get install php5-cgi lighttpd
    $ sudo lighttpd-enable-mod fastcgi

Install the required packages (eibd-clients is optional)

    $ sudo apt-get install linknx eibd-server eibd-clients

Have a look in /etc/defaults/eibd-server and /etc/defaults/linknx, enable
the services and correct the IPs to match your configuration. Of course,
you’ll need a valid XML description of your EIB installation. The
default /etc/linknx/house.xml should give you a pretty extensive
description to start with.

    $ cat /etc/default/eibd-server
    # To enable eibd at startup set this everything != NO
    START_EIBD=YES

    # Options where EIBD should connect to
    # -i usb: for usb or -i ip: for EIBnet/IP routing
    REMOTE_ARGS="-D -T -S -i ipt:192.168.123.124"

    $ cat /etc/default/linknx
    # To enable eibd at startup set this everything != NO
    START_LINKNX=YES

    # Daemon options
    REMOTE_ARGS="--config=/etc/linknx/house.xml"

The options in in eibd-server allow to connect to the eibd device with
ETS and the commands will be forwarded in turn to your module (IP or USB).
For the user friendly interface, you’ll need to extract knxweb in
/var/www/. Since most of the effort for this one is custom in any case
(creating images for your particular installation); there are no packages
for this.

You will then need to modify the background images and design your layout
with the built-in editor (the images will need to be overwritten on your
plug, there is no editor support for that as of yet).

Since knxweb generates a lot of http accesses, you might want to disable
in /etc/lighttpd/lighttpd.conf the access logging module (mod_accesslog).

Sweet Home 3D is a nice cross platform and easy to use 3D modelling tool
I used for the attached screenshot.

