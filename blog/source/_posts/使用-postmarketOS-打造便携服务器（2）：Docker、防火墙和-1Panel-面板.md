---
title: 使用 postmarketOS 打造便携服务器（2）：Docker、防火墙和 1Panel 面板
date: 2026-03-17 22:38:39
tags: 
- Linux
- postmarketOS
categories: 技术
---
作为一台服务器，当然是要装上我最爱的服务器面板 1Panel 社区版了。单开一个页面，说明这里面也有一些坑。

# Docker
使用 pmOS 的一大原因就是支持 Docker，但我想用一键命令安装面板时才发现脚本没法自动安装 Docker，不过 Alpine Linux 也是可以通过包管理器安装 Docker 的，直接安装：

```bash
apk add docker
```

## containerd 引发的坑
这里就要说道说道了。正常情况下，`docker` 软件包依赖的 `docker-engine` 软件包有一个依赖，叫 `containerd`，这个 `containerd` 按理来说都不需要手动配置，然而我发现安装好 Docker 却死活无法启动，使用 `docker info` 命令得知 socket 无法连接，而 socket 是由 `containerd` 提供的。

之后我发现了更有意思的现象，`containerd` 这个服务也是死活无法启动。经过一番研究，我发现 `containerd` 的服务中，`ExecStart` 的值为 `/usr/local/bin/containerd`，但是这个目录下根本没有一个名叫 `containerd` 的文件。

不过没有文件无所谓，既然能通过软件包管理器装上，这个文件肯定在别的地方，直接用 `whereis containerd` 查找，最终得到两个结果：

```bash
root@OnePlus-6:~# whereis containerd
containerd: /usr/bin/containerd /etc/containerd
```

得到的结果中，`/etc/containerd` 是目录，那么 `/usr/bin/containerd` 就是我们要找的文件了。

## systemd 的小坑

既然找到了目标文件，下一步自然是让服务启动这个文件。这里就要提到一个实用特性：可以使用 `systemctl edit` 直接修改已有的服务文件。

然而这个特性也有坑，某些配置项并不是覆盖，而是添加。如果我们将 `ExecStart=/usr/bin/containerd` 直接写入服务文件，那么服务文件就会出现两个 `ExecStart`，从而导致服务解析失败无法启动。要解决这个问题很简单，在写入的行之前再加一个 `ExecStart=` 即可，就像下面这样：

```
### Editing /etc/systemd/system/containerd.service.d/override.conf
### Anything between here and the comment below will become the contents of the drop-in file

[Service]
ExecStart=
ExecStart=/usr/bin/containerd

### Edits below this comment will be discarded


### /usr/lib/systemd/system/containerd.service
......
```

修改完成后，直接使用 `systemctl start containerd` 启动服务，Docker 就可以正常使用了。

# 防火墙和面板

Docker 的问题解决了，下一步就是安装 1Panel 面板。直接使用官方的一键安装指令一步到位后，我发现局域网的其他设备死活连不上网页面板，用端口扫描器扫了一遍发现只有 22 端口是开的。不过使用一加 6 打开 pmOS 自带的浏览器倒是能打开面板，那这就是防火墙的问题了。

查阅[官方资料](https://wiki.postmarketos.org/wiki/Firewall)得知，默认情况下，系统使用 nftables 防火墙阻断除 22 端口外的所有连接，这也包含了使用非标准端口的面板服务。

不过，1Panel 是可以借助 iptables 管理防火墙的，我不太需要 nftables，所以直接关闭并禁用服务：

```
systemctl stop nftables
systemctl disable nftables
```

这样，面板就也能连上了。后续要管理防火墙规则，全部在 1Panel 面板上搞定。
