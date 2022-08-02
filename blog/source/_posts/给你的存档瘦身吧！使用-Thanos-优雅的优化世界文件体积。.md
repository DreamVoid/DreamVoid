---
title: 给你的存档瘦身吧！使用 Thanos 优雅的优化世界文件体积。
date: 2022-08-02 15:53:11
tags:
- Minecraft
- Spigot
---

# 给你的存档瘦身吧！使用 Thanos 优雅的优化世界文件体积。

## 前言
### Thanos 是什么？
Thanos 是一个 PHP 库，由 Aternos 团队开发，可以自动检测并删除 Minecraft 世界中未使用的区块。这可以将世界文件的大小减少 50% 以上，从而释放磁盘空间。

Thanos 是开源的，其源代码可以在 https://github.com/aternosorg/thanos 找到。

### Thanos 会不会误删存档中的建筑等有价值的区块？
**不会**。Thanos 会使用玩家在区块中停留的时间来确定区块是否有价值吗，这可以防止玩家的建筑等重要区块被意外删除，并能使优化过后的世界与大多数插件兼容。

### 我能使用 Thanos 优化基岩版的世界吗？
**不行**。Thanos 仅支持 Java 版的世界格式。

## 开始使用
### 准备工作

要开始使用Thanos，你需要准备以下环境：
* Linux（建议）或 Windows
* PHP 7

本篇教程我将使用以下环境：
* CentOS 7.9
* PHP 7.4（使用宝塔安装）

### 安装 Thanos

首先，使用 `mkdir` 命令创建一个新文件夹，因为安装 Thanos 会生成一些文件，使用文件夹可以避免目录杂乱。下面的命令将会在当前目录新建并自动转到名为“Thanos”的文件夹（下文提到的 Thanos 文件夹均指此处新建的文件夹）：

```bash
mkdir Thanos && cd Thanos
```

然后，执行以下命令安装 Thanos：
```bash
composer require aternos/thanos
```

如果你是首次通过宝塔安装 PHP，则在执行以上命令后会出现这样的红底白字警告：```putenv() has been disabled for security reasons```，此时需要进入宝塔面板的 PHP 管理中删除禁用函数。

你一共要删除这些函数：
* putenv
* proc_open

删除完成后，再次执行安装命令完成 Thanos 的安装。

### 使用 Thanos

关闭 Minecraft 服务器，然后将你要优化的世界文件夹复制到 Thanos 文件夹，执行以下命令：
```bash
./vendor/aternos/thanos/thanos.php 世界文件夹名称 输出文件夹名称
```

例如，我要优化名为“world”的世界，优化后的世界名为“worldfix”，那么我就执行以下命令：
```bash
./vendor/aternos/thanos/thanos.php world worldfix
```

耐心等待一会，当出现 `Removed 0 chunks in 1.95 seconds` 这样的文字说明优化已完成。

同时一个新的文件夹“worldfix”就会生成，这就是优化后的世界。

我们只需要将 Minecraft 服务端的世界文件夹删除，将这个“worldfix”复制到 Minecraft 服务端根目录并重命名为“world”，优化就完成了，现在就可以启动 Minecraft 服务端了。

根据原始的世界文件夹大小，优化所花费的时间也会变长。

## 效果展示

我的 1.19 生存服务器的主世界原来的大小接近 9 GB：

![优化前](/image/before.png)

使用本工具优化以后，大小变为 5.44 GB，再也不怕玩家跑图生成无意义的区块占用磁盘空间了：

![优化后](/image/after.png)

## 注意事项

不要在优化过程中更改世界文件夹或直接在 Minecraft 服务端运行时进行优化，否则**可能会造成不可逆的后果。**

**务必进行备份**，有了备份，你就可以放心的进行世界优化而不必担心优化失败的后果。

如果使用 SSH 连接到远程服务器上进行优化，一定要**保证连接稳定**或使用 `nohup`，否则一旦连接断开，优化进程就会终止，从而可能导致世界文件损坏。

尽量**不优化原版单文件夹存档**，Thanos 为 Spigot 的“一个文件夹就是一个世界”打造，可能无法完美支持原版的“一个文件夹就是全部世界”。