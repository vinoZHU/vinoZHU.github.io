---
title: Linux系统启动流程
date: 2016-09-16 19:40:21
tags: 
- Linux
- Boot
- GRUB
categories: Linux
---
>原文地址:http://www.linux178.com/linux/linuxinit.html

#### Linux系统启动流程概览

```
POST-->BootLoader(MBR)-->Kernel(硬件探测、加载驱动、挂载根文件系统、/sbin/init)
-->init(/etc/inittab:设定默认级别、系统初始化脚本、启动及关闭对应级别的服务、启动终端)
```

#### 第1步

POST（Power On Self Test/上电自检）

打开电源按钮，CPU会把位于CMOS中的BIOS程序加载到内存里面执行，BIOS会探测并识别主板上的所有硬件，然后按照BIOS程序里面设定的启动顺序（1.光驱 2.硬盘 3.软驱 等），它会挨个去这些设备里面找启动设备，一旦找到就停止寻找，如：第一个先从光驱找到，但是没有找到光盘，那么找第二个硬盘，找到硬盘也不一定能启动，要看硬盘是否包含MBR，如果有MBR那就从硬盘启动，如果没有就继续向下寻找，如果一直没有找到可启动的设备，那么本次启动宣告失败。
   
开电之后，CPU就到出厂时指定的内存地址空间（是由内存和CMOS组成）去加载BIOS程序（存储在CMOS里面），BIOS是由一系列的汇编指令组成，用于进行硬件检测（把检测到的结果存储到内存的低地址空间里，是由于BIOS 的寻址能力有限），BIOS首先会探测有几块内存以及其他设备是不是都基本正常，有任何问题就会报警，就无法往下启动，接着去扫描ISA总线和PCI总线去查找各关联到的设备，并且能指挥各硬件完成中断注册和IO端口注册。

#### 第2步

在上一步中，BIOS找到硬盘的MBR（位于硬盘的0磁道0扇区 大小为512字节，该区域不能被分配给任何分区），然后在MBR中寻找BootLoader（目前比较常用有LILO 和 GRUB，LILO已经不常用，BootLoader在MBR所占446字节，所以必须短小精悍，还有16字节是分区表信息，剩下2字节是用来标明该MBR是否有效），然后把BootLoader加载到内存中开始执行，BootLoader主要功能就是从硬盘找到内核文件，把内核文件加载到内存执行。

**GRUB的功能**

1. 选择启动的内核映像或操作系统；
2. 传递参数：e: 编辑模式 b: 引导
3. 基于密码保护 (这个工具 grub-md5-crypt 可以生成 然后放到grub.conf里面 password --md5 密码 )
4. 启用内核映像 传递参数

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