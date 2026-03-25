---
title: 使用 postmarketOS 打造便携服务器（1）：在一加 6 上安装
date: 2026-03-16 19:34:57
tags: 
- Linux
- postmarketOS
categories: 技术
---

# postmarketOS？
简而言之，[postmarketOS](https://postmarketos.org/) 是一个为移动设备移动设备及其他平台的 Linux 发行版，基于 Alpine Linux。相比原厂以及 LineageOS 等还在使用老旧的 4.9 内核，postmarketOS 已经遥遥领先的使用了 Linux 6.16.7，并且还在持续更新，而由于其是完整 Linux，玩玩 Linux 桌面游戏、运行 Docker 等等操作都不在话下。

对于一台备用机而言，与其用 LineageOS 忍受越来越臃肿的国产应用和无理风控，装上 pmOS 变成服务器更能发挥它的价值。

# 为什么是一加 6？
选用一加 6 实为机缘巧合。某日闲逛海鲜市场发现一台 8+128 的一加 6 要价 300 元，尚有闲钱的我正好需要设备安装 SIM 卡，于是果断入手。最初只是使用 LineageOS 短信转发，之后该销卡的销卡，该转 eSIM 的转 eSIM，这部手机也渐渐闲置下来。又正好在网上冲浪研究 WoA 时发现能够安装有良好支持的 pmOS，遂动手刷机，打开了新世界的大门。

使用一加 6 安装 pmOS 有诸多好处。首先是骁龙 845 的可玩性极佳，性能虽然比不上旗舰，但作为服务器还是够用的；其次是一加 6 在 pmOS 属于官方支持的设备（实际上 Main 才是官方支持，但 Main 全是 QEMU，姑且把 Community 也算官方支持吧），可以直接使用官方预构建镜像，也不容易在使用时遇到奇奇怪怪的问题。

# 安装

首先，在 [pmOS 安装页面](https://postmarketos.org/install/)找到 `OnePlus 6`，右侧有带版本号的链接和一个 `edge` 链接，带版本号的链接为稳定版， `edge` 为测试版，我选择使用稳定版的系统。点击进入后，出现 4 个文件夹图标的链接，分别对应不同的桌面环境，想要传统界面可以选用 `gnome-mobile`，而我觉得 `plasma-mobile` 界面更好看，还有 KDE 生态支持（虽然作为一台服务器用处不大），其他两个没有尝试，就就不做评价了。

下载页面展示的两个 `xz` 格式的压缩包，解压后得到两个 `img` 镜像。将解锁后的手机重启到 Bootloader 模式，依次执行下面几条命令：

```bash
fastboot erase dtbo
fastboot flash boot [文件名带 boot 的镜像]
fastboot flash userdata [不带 boot 的镜像]
```

清除 dtbo 之后就无法再启动和 Android 有关的系统（包括 TWRP），日后想用回 Android 相关的操作系统，则需要先进入 Bootloader 刷入 dtbo 才能正常启动。

最后，使用 `fastboot reboot` 就得到了一个搭载基于 Alpine Linux 的 postmarketOS 了。

# 特性

虽说 pmOS 基于 Alpine Linux，但有一些不符合 Alpine Linux 的特性。比如 Alpine Linux 通常使用 OpenRC，但 pmOS 使用的是 systemd。

pmOS 同样也是有镜像的，在清华大学开源软件镜像站可以同时找到 [Alpine Linux](https://mirrors.tuna.tsinghua.edu.cn/help/alpine/) 和 [pmOS](https://mirrors.tuna.tsinghua.edu.cn/help/postmarketOS/) 的镜像。

# 坑

记录两个折腾 pmOS 遇到的坑。可能不是坑，可能只是我不会配置。

## 多重引导

最开始，我打算遵循[官方教程](https://wiki.postmarketos.org/wiki/OnePlus_6_(oneplus-enchilada)/Dual_Booting_and_Custom_Partitioning)使用多系统，在手机上使用 UEFI 引导多个操作系统让我很是心动，我都已经准备好同时使用 Android、Windows 和 pmOS 了。

然而我发现我是在做梦。首先我发现，通过 UEFI 引导的 pmOS，无法识别到绝大多数包括显卡在内的驱动，到最后比 QEMU 还卡。其次，这玩意只能引导 pmOS 本身，其他操作系统是没办法正常进系统的。

最后在经过几个 9008 之后，我放弃了多引导的美梦。

## 分区

说到了 9008，就不得不谈到分区了。官方教程还提到可以自定义分区大小，将无用的 `system` 等分区全部删掉扩大可用空间。官方教程使用的是 TWRP 删除分区，且系统要使用 pmbootstrap 构建和刷入。最后在搞炸好几次分区表之后，我也放弃了扩大分区的想法。