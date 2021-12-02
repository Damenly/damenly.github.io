---
title: "Qnap 453Dmini install archlinux 威联通 安装archlinux"
date: 2021-12-02T10:55:14+08:00
draft: false
---

Since 453Dmini is sold in only China. So I will write it in Chinese.

之前手上的星际大陆矿机感觉太笨重了，正好双11，便入了威联通453Dmini, 因为我本来
用的是archlinux，而又由于该型号是中国区特供，所以我担心是否 能够从U盘启动等问题。
很多人会问为什么要装linux而不用自带的威联通系统，原因是因为：我喜欢你管得着？
archlinux有最新的软件，不比封闭的方便？因此此文章提供给不满意封闭系统的人，谢谢。

## 启动问题
到手之后发现默认进入的是qnap系统，此时注意不要插任何HDD/SSD,否则会抹掉磁盘
上的数据！直接插上archlinux的U盘，然后开机时看到图标，按f2进入bios，修改
第一启动项目为U盘。点Save and exit之后进入archlinux。

mount 上qnap的mmcblk0一看，内核是基于4.14 ubuntu的：
![](/qnap_boot.jpg)

为了防止qnap的启动启动之后抹掉盘上的数据，我们需要抹掉mmblk0上的数据。
注意首先用 dd if=/dev/mmcblk0 备份qnap的系统。

这边列下我用只有3G的mmcblk0左为archlinux的boot和EFI分区，之后按照archlinux官方
的书册安装就可以了。如果是已经有linux的系统，注意手动更新下rootfs里面的grub
/etc/fstab。重启拔掉U盘，应该就可以从原来的磁盘启动了。

```
mmcblk0      179:0    0   3.7G  0 disk
├─mmcblk0p1  179:1    0   256M  0 part /boot/EFI
└─mmcblk0p2  179:2    0   3.4G  0 part /boot
```

## 453Dmini sata控制器写入问题（可能）
我比较懒，直接贴上我提的ticket的描述好了，至今为止官方未回复我:


主题：ata2.00: exception Emask 0x10 SAct 0x7ff8000f SErr 0x400101 action 0x6 frozen

Recently, I bought a 453Dmini to use as my nas. Instead of QTS, I manually install Archlinux on it but something looks weird.

The first machine booted normally and I inserted and mounted two disks (btrfs raid 0, no smartctl error) into the nas.

While calling umount, there are many ATA commands errors in dmesg. Then I found the link https://forum.qnap.com/viewtopic.php?t=133717.

I thought that it may be a hardware issue so I returned it and bought a new one which arrived today. And the issue happened again.

I tried many combinations of my three disks (two btrfs raid0 disks and one 2.5-inch ssd containing Archlinux). Only When two raid0 disks are in

slots 1 and 2, read, write and umount work well??? And since 453dmini is only supplied in the Chinese market, I can't find anymore on the Internet.

My question: What's the cause of the issue? It's something like a hardware design or hardware issue?

I know you may ask me to use QTS to verify this but it requires me to reformat my only existing two disks.


![](/qnap_disk_error.jpg)

详细的dmesg下载
[dmesg](/shared/dmesg)


## 网卡问题
默认没网卡。453dmini需要手动 modeprobe r8169
