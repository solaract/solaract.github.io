---
layout: post
title: MBR与GPT
tags: [Linux,Vbird Linux]
---
### 硬件在Linux中的文件名
在Linux系統中，每个硬件都被当作一个文件
| 硬件 | Linux中的文件名 |
| ---- | ---- |
|SCSI/SATA/USB硬盘|/dev/sd[a-p]|
|USB FLASH|/dev/sd[a-p]（与SATA相同）|
|VirtI/O界面|/dev/vd[a-p] (用于虚拟机内)|
|软驱|/dev/fd[0-1]|
|打印机|/dev/lp[0-2] (25针打印机)<br>/dev/usb/lp[0-15] (USB)|
|鼠标|/dev/input/mouse[0-15] (通用)<br>/dev/psaux (PS/2界面)<br>/dev/mouse (当前鼠标)|
|CDROM/DVDROM|/dev/scd[0-1] (通用)<br>/dev/sr[0-1] (通用，CentOS 较常见)<br>/dev/cdrom (当前 CDROM)|
|磁带机|/dev/ht0 (IDE 界面)<br>/dev/st0 (SATA/SCSI界面)<br>/dev/tape (当前磁带)|
|IDE硬盘|/dev/hd[a-d] (旧系统)|
- 一般实体机器使用的都是/dev/sd[a-]的文件名，虚拟机环境下，为了加速可能会使用/dev/vd[a-p]
- SATA/USB/SAS等磁盘都是使用SCSI模组驱动，因此文件名都是/dev/sd[a-p]形式，具体文件名由Linux核心检测到磁盘的顺序决定。

### MSDOS（MBR）与GPT磁盘分割表
#### MBR: Master Boot Record 主分区引导记录

> CHS寻址模式将硬盘划分为磁头（Heads）、柱面(Cylinder)、扇区(Sector)
> - 磁头(Heads)：每张磁片的正反两面各有一个磁头，一个磁头对应一张磁片的一个面。因此，用第几磁头就可以表示数据在哪个磁面。
> - 柱面(Cylinder)：所有磁片中半径相同的同心磁道构成“柱面"，意思是这一系列的磁道垂直叠在一起，就形成一个柱面的形状。简单地理解，柱面数=磁道数。 
> - 扇区(Sector)：将磁道划分为若干个小的区段，就是扇区。虽然很小，但实际是一个扇子的形状，故称为扇区。每个扇区的容量为512字节。
> ![硬盘图示](..\assets\img\2021-01-14-Linux-study-1\CHS.png)
> 
开机管理程序和硬盘分区表都存放硬盘的第一个扇区（512 bytes）
- 主分区引导记录MBR：安装开机管理程序的地方，446 bytes（0000H - 01BDH）
- 硬盘分区表DPT（Disk Partition Table）：记录硬盘分割状态，64 bytes（01BEH - 01FDH）
- magic number：55AAH，2 bytes（01FEH – 01FFH）

![MBR结构](..\assets\img\2021-01-14-Linux-study-1\MBR.png)
分区表仅有64 bytes容量，最多仅能有四组记录区（16 bytes），每组记录了该区段的起始与结束的磁柱号码
> 为文件名为/dev/sda的硬盘分区时，Linux系统中分区的文件名在硬盘文件名后加数字（与分区所在位置有关），例：
> - /dev/sda1
> - /dev/sda2

这四个记录区被称为主（primary）或扩展（extended）分区
- 分区只是针对64 bytes的硬盘分区表进行设置
- 硬盘预设分区表仅能写入四组分区记录
- 这四组分区记录称为主primary）或扩展（extended）分区
- 分区的最小单位通常为磁柱（cylinder）
- 系统写入硬盘时，一定会参考硬盘分区表，才能对分区进行资料整理

最多只能分割出4个主分区，需要更多分区时，可以将一个记录区作为扩展分区指向额外区块来记录分区，称为逻辑分区（logical partition）
> /dev/sda1~4文件名保留给主或扩展分区，逻辑分区从/dev/sda5开始

- 扩展分区最多只能有一个
- 能够被格式化后存取数据的分区为主分区和逻辑分区，扩展分区无法格式化

缺点：
1. 作业系统无法识别2.2T以上的硬盘容量
2. MBR仅有一个区块，若被破坏，经常无法或很难救援
3. MBR内存放开机管理程序的区块仅446 bytes，无法容纳较多程序代码



#### GPT: GUID partition table GUID分区表
> LBA（Logical Block Address，逻辑区块地址）将硬盘容量空间逻辑映射成线性编号的多个扇区，一个序数确定一个唯一的扇区，第一个LBA编号为LBA0
> 
> 用C表示当前柱面号，H表示当前磁头号，S表示当前扇区号，CS表示起始柱面号，HS表示起始磁头号，SS表示起始扇区号，PS表示每磁道有多少个扇区，PH表示每柱面有多少个磁道，LBA序号的计算公式如下：
>
>       LBA = ( C – CS ) * PH * PS + ( H – HS ) * PS + ( S – SS )

GPT使用34个LBA区块记录分区，硬盘最后33个LBA对分区信息进行备份

![GPT分区表](..\assets\img\2021-01-14-Linux-study-1\gpt_partition_1.jpg)

- LBA0（MBR相容区块）
  
  存放MBR信息（开机程序+分区表），当不支持GPT的分区工具试图对硬盘进行操作时，能够支持MBR方式启动；使用GPT时，在分区表中放入一个特殊标记
- LBA1（GPT表头）
  主要定义了分区表中项目数、每项大小、硬盘容量等信息。分区表头还记录了这块硬盘的分区表和分区表头的位置（总是LBA1）与大小，也包含了备份分区表头和备份分区表的位置和大小信息（LBA-1~LBA-34）。同时还储存着它本身和分区表的CRC32校验码。当出现错误时可以从备份中恢复整个分区表
- LBA2-33（分区表）
  每个LBA可以记录4个分区，最多可有4 * 32=128个分区。每个分区记录使用128 bytes空间，除相关记录外，使用64 bits记录分区起止LBA号（共64 bits * 2），因此GPT支持的最大容量为264 * 512 bytes = 8 ZB（扇区大小为512 bytes时）