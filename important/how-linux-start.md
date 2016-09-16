---
title: Linux系统启动流程
date: 2016-09-16 19:40:21
tags: 
- Linux
- Boot
- GRUB
categories: Linux
---
>本文在原文地址:http://www.linux178.com/linux/linuxinit.html的基础上进行了些许修改

#### Linux系统启动流程概览

```
POST（Power On Self Test/上电自检）-->BootLoader(MBR)-->Kernel(硬件探测、加载驱动、挂载根文件系统、/sbin/init)
-->init(/etc/inittab:设定默认级别、系统初始化脚本、启动及关闭对应级别的服务、启动终端)
```

#### 第1步

计算机电源接通后，CPU默认执行`0ffffh:0000h`处的指令（8086是这样，386应该类似），而此内存地址应该存放的就是`BIOS ROM`。系统将有一个对内部各个设备进行检查的过程，这是由一个通常称之为POST（Power On Self Test/上电自检）的程序来完成，这也是BIOS程序的一个功能。完整的自检包括了对CPU、640K基本内存、1M以上的扩展内存、ROM、主板、CMOS存贮器、串并口、显示卡、软硬盘子系统及键盘的测试。在自检过程中若发现问题，系统将给出提示信息或鸣笛警告。

如果没有任何问题，完成自检后BIOS将按照系统CMOS设置中的启动顺序搜寻软、硬盘驱动器及CDROM、网络服务器等有效的启动驱动器 ，读入操作系统引导记录（BootLoader），然后将系统控制权交给引导记录，由引导记录完成系统的启动。如果一直没有找到可启动的设备，那么本次启动宣告失败。

#### 第2步

在上一步中，BIOS找到硬盘的MBR（位于硬盘的0磁道0扇区 大小为512字节，该区域不能被分配给任何分区），然后在MBR中寻找BootLoader（目前比较常用有LILO 和 GRUB，LILO已经不常用，BootLoader在MBR所占446字节，所以必须短小精悍，接下来64字节是分区表信息，最后2字节是用来标明该MBR是否有效），然后把BootLoader加载到内存中开始执行，BootLoader主要功能就是从硬盘找到内核文件，把内核文件加载到内存执行。

MBR的三部分如下:

- 第1-446字节：BootLoader。
- 第447-510字节：分区表（Partition table）。
- 第511-512字节：主引导记录签名（0x55和0xAA）。

扩充：

分区表的长度只有64个字节，里面又分成四项，每项16个字节。所以，一个硬盘最多只能分四个一级分区，又叫做"主分区"。
每个主分区的16个字节，由6个部分组成：

- 第1个字节：如果为0x80，就表示该主分区是激活分区，控制权要转交给这个分区。四个主分区里面只能有一个是激活的。
- 第2-4个字节：主分区第一个扇区的物理位置（柱面、磁头、扇区号等等）。
- 第5个字节：主分区类型。
- 第6-8个字节：主分区最后一个扇区的物理位置。
- 第9-12字节：该主分区第一个扇区的逻辑地址。
- 第13-16字节：主分区的扇区总数。

疑问? 内核文件在哪里？GRUB是怎么找到内核文件？

内核文件（vmlinuz-2.6.18-308.el5）是位于/boot分区下（在我们给硬盘分区的时候都会把/boot单独分区），这时又有疑问了，这时候/都没有被挂载，又如何从硬盘上找到内核文件？
![](/images/linux/how-linux-start-0.png)

这时看GRUB的配置文件/boot/grub/grub.conf 可以看到 root (hd0,0)，这一行实际上就是指定boot目录所在的位置
而kernel /vmlinuz-2.6.18-308.el5 ro root=LABEL=/ 这里指定的是内核文件所在的位置，而前面的/并不是真正的根，而是指的是boot目录所在的位置，那么其全路径为(hd0,0)/vmlinuz-2.6.18-308.el5,而这里的(hd0,0)指的是第1个硬盘的第1个分区，GRUB在识别硬盘的时候都是识别为hd开头的。

总结: GRUB 不是通过文件系统来找内核文件的，因为这时候内核还没有启动所以也不存在什么文件系统，而是直接访问硬盘的第1个硬盘第1个分区（MBR里面存在分区表）的来找到内核文件。

这时候又有个问题 GRUB是怎么识别分区表中这些分区的文件系统的？ 且看/boot/grub目录下的文件:

```shell
[root@server1 grub]# ll
总计 257
-rw-r--r-- 1 root root     63 2013-01-05 device.map
-rw-r--r-- 1 root root   7584 2013-01-05 e2fs_stage1_5
-rw-r--r-- 1 root root   7456 2013-01-05 fat_stage1_5
-rw-r--r-- 1 root root   6720 2013-01-05 ffs_stage1_5
-rw------- 1 root root    562 2013-01-05 grub.conf
-rw-r--r-- 1 root root   6720 2013-01-05 iso9660_stage1_5
-rw-r--r-- 1 root root   8192 2013-01-05 jfs_stage1_5
lrwxrwxrwx 1 root root     11 2013-01-05 menu.lst -> ./grub.conf
-rw-r--r-- 1 root root   6880 2013-01-05 minix_stage1_5
-rw-r--r-- 1 root root   9248 2013-01-05 reiserfs_stage1_5
-rw-r--r-- 1 root root  55808 2009-03-13 splash.xpm.gz
-rw-r--r-- 1 root root    512 2013-01-05 stage1
-rw-r--r-- 1 root root 104988 2013-01-05 stage2
-rw-r--r-- 1 root root   7072 2013-01-05 ufs2_stage1_5
-rw-r--r-- 1 root root   6272 2013-01-05 vstafs_stage1_5
-rw-r--r-- 1 root root   8904 2013-01-05 xfs_stage1_5
```

其实GRUB启动是分阶段的：

第1个阶段  BIOS加载MBR里面的GRUB（属于第1阶段的文件），由于只有GRUB只占用446字节所以不能实现太多的功能，所以就有此阶段里面的文件来加载第1.5阶段的文件（/boot/grub下的文件）。

第1.5个阶段 这个阶段里面的就是加载识别文件系统的程序，来识别文件系统，不加载就无法识别文件系统，进而就找不到boot目录，由于GRUB是无法识别LVM，所以你不能把/boot分区设置为LVM，所以必须要把/boot单独分区。

第2个阶段 这里面才是正在的开始寻找内核的过程，然后是启动内核。

#### 第3步

在上一步中，GRUB成功找到内核文件，并把内核加载到内存，同时把/boot/initrd-2.6.18-308.el5.img这个文件也加载进来，这个文件是做什么的呢？

那么首先看看内核在这一步骤里面做的事情：

1. 探测硬件
2. 加载驱动
3. 挂载根文件系统
4. 执行第一个程序/sbin/init

BIOS检查硬件，而内核是会初始化硬件设备，那么首先会探测硬件（第1步），知道是什么硬件了就该加载硬件驱动程序（第2步），不然是没办法指挥着硬件工作的，关键是内核去哪里找驱动程序（驱动程序是硬盘上，是内核模块.ko存在的）而此时根文件系统还没有挂载上，怎么办？ 那可以②③对调，先挂载根文件系统，然后再加载驱动，那此时又有问题了，我不加载驱动程序又如何驱动着硬盘工作呢？ 这个陷入了是先有蛋还是有先鸡的问题了，这该如何解决？

这时候 这个文件/boot/initrd-2.6.18-308.el5.img（该文件是一个.gz的压缩文件） 就派上用场了，这个文件也是被GRUB加载内存当中，构建成一个虚拟的根文件系统，这个文件里面包含有硬件驱动程序（），这个文件是可以展开如下操作：

```shell

[root@server1 boot]# cp initrd-2.6.18-308.el5.img ~
[root@server1 boot]# cd
[root@server1 ~]# ls
initrd-2.6.18-308.el5.img
[root@server1 ~]# mv initrd-2.6.18-308.el5.img initrd-2.6.18-308.el5.img.gz
[root@server1 ~]# gzip -d initrd-2.6.18-308.el5.img.gz
[root@server1 ~]# ls
initrd-2.6.18-308.el5.img
[root@server1 ~]# file initrd-2.6.18-308.el5.img
initrd-2.6.18-308.el5.img: ASCII cpio archive (SVR4 with no CRC) 可以看到此时是一个cpio的归档文件
[root@server1 ~]# mkdir test
[root@server1 ~]# mv initrd-2.6.18-308.el5.img  test
[root@server1 ~]# cd test
[root@server1 test]# ls
initrd-2.6.18-308.el5.img
[root@server1 test]# cpio -id < initrd-2.6.18-308.el5.img  利用cpio来展开该文件
12111 blocks
[root@server1 test]# ls
bin  dev  etc  init  initrd-2.6.18-308.el5.img  lib  proc  sbin  sys  sysroot
[root@server1 test]# mv initrd-2.6.18-308.el5.img ../
[root@server1 test]# ls  可以看到这不就是跟真实的根很像吗
bin  dev  etc  init  lib  proc  sbin  sys  sysroot
[root@server1 test]# ls lib/    可以看到这目录下包含了ext3.ko的内核模块，该模块就可以驱动着硬盘进行工作了
ata_piix.ko            dm-mod.ko              ext3.ko                mptbase.ko             scsi_mod.ko
dm-log.ko              dm-raid45.ko           firmware/              mptscsih.ko            scsi_transport_spi.ko
dm-mem-cache.ko        dm-region_hash.ko      jbd.ko                 mptspi.ko              sd_mod.ko
dm-message.ko          ehci-hcd.ko            libata.ko              ohci-hcd.ko            uhci-hcd.ko
[root@server1 test]#
```
至此内核利用虚拟的根文件系统的ext3.ko内核模块，驱动了硬盘，然后挂载了真正的根文件系统，那么此时虚拟的根文件系统是否还有作用，它还可以挂载/proc文件系统等操作。

此阶段中最后一个步骤 由内核来启动第一个程序/sbin/init，启动好之后剩下的工作就有init进程来完成了。

#### 第4步

init进程首先会读取/etc/inittab文件，根据inittab文件中的内容依次执行

设定系统运行的默认级别（id:3:initdefault:）
执行系统初始化脚本文件（si::sysinit:/etc/rc.d/rc.sysinit）
执行在该运行级别下所启动或关闭对应的服务（l3:3:wait:/etc/rc.d/rc 3）
启动6个虚拟终端:
     
- l0:0:wait:/etc/rc.d/rc 0
- l1:1:wait:/etc/rc.d/rc 1
- l2:2:wait:/etc/rc.d/rc 2
- l3:3:wait:/etc/rc.d/rc 3
- l4:4:wait:/etc/rc.d/rc 4
- l5:5:wait:/etc/rc.d/rc 5
- l6:6:wait:/etc/rc.d/rc 6