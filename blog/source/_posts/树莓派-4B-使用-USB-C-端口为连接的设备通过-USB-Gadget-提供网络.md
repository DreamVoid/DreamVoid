---
title: 树莓派 4B 使用 USB-C 端口为连接的设备通过 USB Gadget 提供网络
date: 2026-06-27 19:37:31
tags: 
- OpenWRT
- 树莓派
categories: 技术
---

有时候只有一根用于联网的网线，想利用树莓派通过 USB 和无线网卡为下游设备提供网络，在此记录操作流程，以供备用。

# 树莓派上的配置

树莓派使用的系统为 ImmortalWrt 官方版，通过板子上的 USB-C 端口与电脑连接。

## 修改启动配置

树莓派的 boot 分区挂载在 /boot 目录下，在这个目录中需要修改两个文件。

config.txt 文件，将 `dtoverlay=dwc2` 添加到 `[all]` 节中。完整文件内容示例如下：
```
################################################################################
# Bootloader configuration - config.txt
################################################################################

################################################################################
# For overclocking and various other settings, see:
# https://www.raspberrypi.com/documentation/computers/config_txt.html
################################################################################

# OpenWrt config
include distroconfig.txt

[pi2]
dtparam=i2c_arm=on,audio=on

[pi3]
dtparam=i2c_arm=on,audio=on

[pi4]
dtparam=i2c_arm=on,audio=on

[all]
# Place your custom settings here.
dtoverlay=dwc2
```

cmdline.txt 文件中，将 `modules-load=dwc2,g_ether` 添加到 `rootwait` 之后。完整文件内容示例如下：
```
console=tty1 console=serial0,115200 root=PARTUUID=cbc8ff3b-02 rootfstype=squashfs,ext4 rootwait modules-load=dwc2,g_ether
```

## 修改启动脚本

LuCI 界面可以在登录后转到“系统”->“启动项”->“本地启动脚本”，终端可以直接修改 `/etc/rc.local` 文件。在这个文件中添加 `modprobe g_ether`。完整文件内容示例如下：
```
# Put your custom commands here that should be executed once
# the system init finished. By default this file does nothing.
modprobe g_ether
exit 0
```

## 安装软件包

安装 `kmod-usb-dwc2` 和 `kmod-usb-gadget-cdc-composite`。

## 将网卡加入网桥

这一步是为了让连接的设备通过树莓派的 usb0 网卡加入 LAN，从而能够让连接的设备像接入网线一样上网。如果有其他用途，可忽略这一步，将 usb0 网卡根据自己的喜好操作。

LuCI 界面中，转到“网络”->“接口”->“设备”，找到 `br-lan`，点击右侧的“配置”按钮，在“网桥端口”中，最下方的自定义编辑框输入“eth0”后回车，然后点击“保存”，最后点击页面底部的“保存并应用”。

以上步骤全部完成后，重启树莓派。

# Windows 上的配置

如果树莓派的配置没有问题，重启树莓派之后，Windows 应该会提示接入了新的设备，在任务管理器可以看到“端口”中多出了一个“USB 串行设备”，这就是连接的树莓派。

要将这个设备变为网卡设备，需要安装一个适用于树莓派的 RNDIS 驱动。[我在 GitHub 上准备好了这个驱动文件](https://github.com/DreamVoid/DreamVoid/blob/main/archive/RPI%20Driver%20OTG.zip)，下载之后解压备用。

在设备管理器中，右键选择这个串行设备，点击“更新驱动程序”，选择“浏览我的电脑以查找驱动程序”，点击“浏览”，之后选中解压的驱动包，点击“下一步”安装驱动后，设备名称就会变成“USB Ethernet/RNDIS Gadget”，此时系统也会多出一个网卡，可通过这个网卡与树莓派联网。