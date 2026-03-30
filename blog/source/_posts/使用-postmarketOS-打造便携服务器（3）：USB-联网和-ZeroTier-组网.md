---
title: 使用 postmarketOS 打造便携服务器（3）：USB 联网和 ZeroTier 组网
date: 2026-03-30 11:02:38
tags: 
- Linux
- postmarketOS
- ZeroTier
categories: 技术
---

# USB 联网
我个人使用的联网方案是安装了 ImmortalWrt 的树莓派 4b，使用树莓派的好处多多，这里不做展开。我已经提前在 ImmortalWrt 上装好了 `kmod-usb-net-rndis` 软件包，这样就能够使用 RNDIS 看到手机的 USB 共享网络，当然也包括安装了 postmarketOS 的一加 6。

## 配置
要配置一加 6 通过 USB 接入树莓派联网，只需要配置自带的 `usb0` 网卡即可。幸运的是，只需要用 1 条命令即可完成。

在 pmOS 的 Shell 中，以 root 身份执行命令 `nmtui`，即可进入 NetworkManager 的管理界面，用方向键选择 `Edit a connection`（`编辑连接`），找到 `USB Networking`，选择右侧的 `Edit...`（`编辑...`），将 `IPv4 CONFIGURATION` 右侧的 `Manual`（`手动`）改为 `Automatic`（`自动`），之后移除唯一的一条 `Addresses`，保存即可。

在 ImmortalWrt 网页中，依次选择 “网络” -> “接口” -> “设备”，找到 `br-lan`，点击右侧的 “配置...”按钮，将新出现的 `eth1` 加入网桥。如果 `eth1` 被其他网卡占用，就会变成 `eth2`、`eth3`...以此类推。

如果不出意外，此时 DHCP 服务器会给 pmOS 发放 IP，就可以通过 USB 连接网络。至于无线网络？作为防失联的备用联网方案倒还行，只能说用过树莓派 4b 无线网卡的懂的都懂。

## 持久化
新版本的 pmOS 对网络连接添加了持久化配置，只在 `nmtui` 设置会导致重启后还原为先前的配置。

持久化目录位于 `/etc/NetworkManager/system-connections/`，这里有一个 `USB Networking.nmconnection` 文件，修改文件中 `[ipv4]` 和 `[ipv6]` 的 `method` 为 `auto`，然后删除其他无用的配置即可。如果图方便，也可以选择直接删除整个文件，但我不会删的，我可不想哪天网络配置挂了还要用手机屏幕来配置网络。

# ZeroTier 组网
Alpine Linux 的 ZeroTier 比较特殊，官方安装脚本无法安装，包管理器也无法安装。原因不是我们要讨论的，既然无法用现成的方法安装，就只能回归最传统的方式：编译安装。

ZeroTier 是开源软件，可以在 https://github.com/zerotier/ZeroTierOne 找到源码。我选择了 1.16.0 的 tag 作为此次要使用的源码。其实写文章的时候 1.16.1 已经出了，但不知道为什么没有发版本，而是以分支形式存在，不知以后是否还会变化，因此不使用 1.16.1。

## 下载源码并编译
下载 1.16.0 的源码并解压：
```bash
wget https://github.com/zerotier/ZeroTierOne/archive/refs/tags/1.16.0.zip
unzip 1.16.0.zip
```

安装编译需要的软件包：
```bash
apk add util-linux pkgconfig openssl-dev libffi cargo linux-headers g++ gcc make
```

进入源码目录编译：
```bash
cd ZeroTierOne-1.16.0/
make -j$(nproc)
```

如果不出意外，编译完成后，在当前目录下有一个 `zerotier-one` 文件，这就是 ZeroTier 的主程序，还有 `zerotier-idtool` 和 `zerotier-cli` 两个链接到主程序的命令行文件。

## 运行服务
将编译得到的三个文件移动到 `/usr/sbin` 目录下，和官方安装方式的文件位置保持一致：
```bash
mv zerotier-one /usr/sbin/
mv zerotier-cli /usr/sbin/
mv zerotier-idtool /usr/sbin/
```

然后编写服务文件，存放在 `/lib/systemd/system/zerotier-one.service`，以下内容和官方安装方式创建的文件也是一样的：
```
[Unit]
Description=ZeroTier One
After=network-online.target network.target
Wants=network-online.target

[Service]
ExecStart=/usr/sbin/zerotier-one
Restart=always
KillMode=process

[Install]
WantedBy=multi-user.target
```

创建完成后，使用 `systemctl daemon-reload` 加载服务文件，然后使用 `systemctl enable zerotier-one` 设为开机自启。

运行之前，还要做一个准备工作，让 `tun` 模块自动加载，ZeroTier 的网络接口才会出现。自动加载也很简单，编辑 `/etc/modules` 文件，在末尾添加 `tun` 即可。像下面这样：
```
af_packet
ipv6
tun
```

如果还没有重启，此时可以直接 `reboot` 重启，这样之前做的修改就可以同时生效，也能验证持久化是正常的。重启之后，使用 `systemctl status zerotier-one`，应该就能看见服务正常运行，之后按照正常使用 ZeroTier 的方法即可。