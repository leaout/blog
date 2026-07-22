---
title: M401A机顶盒刷机安装armbian
date: 2024-12-15 22:26:08
tags:
---
[魔百盒](https://www.znds.com/bbs-342-1.html)M401A原本只是一款普通的机顶盒，但经过刷入开源操作系统armbian，它将获得全新的生命力和无限的可能性。本[教程](https://www.znds.com/jc/)将一步步指导您完成整个[刷机](https://www.znds.com/bbs-102-1.html)过程，从准备阶段到具体操作，让您轻松掌握刷机技巧，让魔百盒M401A焕发出全新的光彩。

**一、硬件与树莓派对比**

[![img](https://app.yinxiang.com/images/file-generic.png)36.5 KB
](https://app.yinxiang.com/shard/s68/res/16b06889-00ad-4b3f-89c9-c1bd3a745877)



![img](https://app.yinxiang.com/shard/s68/res/1153e80b-8af7-4ee4-8183-9aede23319ae.gif)

**二、 检查自己的M401A是哪种类型的版本**

[魔百盒怎么样刷系统 魔百盒311-1a YST 刷Armbian系统教程分享](https://www.znds.com/tv-1246318-1-1.html) 上面的github issue主要介绍了如何分辨M401A同型号但是不同结构。主要分为两种类型三种结构：

> 类型A
>
> - 强版本
>
> 2GHz的EMMC参数和1.8GHz的CPU参数，dtb驱动 ddr4内存是scy;emmc标识为sec 137字样;板子的logo文字印刷在靠近蓝牙芯片处.
>
> - 弱版本
>
> 1GHz的EMMC参数和1.7GHz的CPU参数，dtb驱动 总体同上面强体质版本，emmc标识为silicongo.
>
> 类型B ddr4内存是CXMT;emmc标识为silicongo字样;板子的logo文字印刷在靠近USB接口处.

![img](https://app.yinxiang.com/shard/s68/res/1153e80b-8af7-4ee4-8183-9aede23319ae.gif)

**三、刷机前准备**

开始刷机前需要准备一些工具：

1. 电视盒子、U盘、键盘、显示器、HDMI线

*复制代码*

1、下载Armbian系统 https://github.com/ophub/amlogic-s9xxx-armbian/releases

[![img](https://app.yinxiang.com/images/file-generic.png)9.4 KB
](https://app.yinxiang.com/shard/s68/res/b180f0ed-06ad-4a30-ae72-53136c6f175e)



进入上面的开源库，找到Armbian_23.02.0_amlogic_s905l3a_jammy_6.1.18_server_2023.03.16.img.gz类似这样格式的安装包，下载安装即可。

2、将Armbian系统写入U盘

我这边使用的时rufus工具，下载地址：[rufus-3.21](https://www.aliyundrive.com/s/h3Gpfcyjwko)[_](https://www.aliyundrive.com/s/h3Gpfcyjwko)[BETA.exe](https://www.aliyundrive.com/s/h3Gpfcyjwko) 提取码: kc09

首先选择使用的U盘设备 选择刚刚下载的Armbian系统 点击开始，U盘会要求格式化，确定后会自动写入U盘

[![img](https://app.yinxiang.com/images/file-generic.png)23.1 KB
](https://app.yinxiang.com/shard/s68/res/bfe27add-8716-4697-817f-0a3e90781465)



3、写入U盘后参数更改

成功写入U盘后打开U盘内的uEnv.txt，修改使用的dtb文件

[![img](https://app.yinxiang.com/images/file-generic.png)12.9 KB
](https://app.yinxiang.com/shard/s68/res/b4235b0d-118e-4ce5-a835-bb26eb137dab)



1. FDT=/dtb/amlogic/meson-g12a-s905l3a-e900v22c.dtb

   修改为

   FDT=/dtb/amlogic/meson-g12a-s905l3a-m401a.dtb

*复制代码*

![img](https://app.yinxiang.com/shard/s68/res/1153e80b-8af7-4ee4-8183-9aede23319ae.gif)

**四、开始刷机**

1、盒子从U盘进行启动 拆开盒子，找到主板后面的重置按钮 插上U盘，先按住重置按钮，插上电源，过5秒左右松开按钮，等待看是否已经进入刷armbian系统的流程

2、设置账户 进入armbian系统后会加载一会程序，然后提示需要设置root密码，还有设置时区

3、Armbian写入EMMC 输入命令armbian-install

提示选择安装设备型号

[![img](https://app.yinxiang.com/images/file-generic.png)36.9 KB
](https://app.yinxiang.com/shard/s68/res/89eff3ed-0b89-4225-9e7f-23bab2cd2882)



这边选择306 M401A

FDTFILE 选择meson-g12a-s905l3a-m401a.dtb 选择文件系统

[![img](https://app.yinxiang.com/images/file-generic.png)13.0 KB
](https://app.yinxiang.com/shard/s68/res/235586c4-d50b-48ea-96b2-453a38cec500)



这边选择**1 **ext4 接下来系统会自动进行初始化工作。。。

4、重启系统

系统提示Successful installed, please unplug the USB，re-insert the power supply to start the armbian.。表示已经安装完成，但是现在不要拔U盘，输入reboot 进行系统重启。

重启完成后可以断电拔出U盘，然后通电启动进入系统。