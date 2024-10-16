---
title: 自建 ZeroTier Planet
date: 2024-10-16 21:22:25
tags: ZeroTier
categories: 技术
---
ZeroTier 是一个好的组网软件，搭配上自建的 planet 根服务器能让体验更上一层楼。

# 搭建 planet
首先克隆 ZeroTierOne 的源码：

```bash
git clone https://github.com/zerotier/ZeroTierOne.git
```

之后，进入 `attic/world` 目录，目录下的 `mkworld.cpp` 的文件即是需要修改的文件。

找到文件中的以下内容（一般在 86 行）：
```
	roots.push_back(World::Root());
	roots.back().identity = Identity("3a46f1bf30:0:76e66fab33e28549a62ee2064d1843273c2c300ba45c3f20bef02dbad225723bb59a9bb4b13535730961aeecf5a163ace477cceb0727025b99ac14a5166a09a3");
	roots.back().stableEndpoints.push_back(InetAddress("185.180.13.82/9993"));
	roots.back().stableEndpoints.push_back(InetAddress("2a02:6ea0:c815::/9993"));
```

此处定义了 ZeroTier 官方的 4 个根服务器地址，按照相同的格式修改 IP 和端口即可。

其中，`identity` 的值可以通过以下命令获取：`cat /var/lib/zerotier-one/identity.public`

编辑完成后，在相同目录下运行编译命令 ` ./build.sh && ./mkworld`。

如果出现报错 `c++: command not found`，则首先运行 `apt install g++` 安装编译工具。

最后得到的产物 `world.bin` 就是 planet 文件，使用命令 `mv world.bin planet` 将其重命名。

# 搭建 ztncui 管理面板

首先克隆 ztncui 的源码：

```bash
git clone https://github.com/key-networks/ztncui
```

之后使用 Node 安装 ztncui：

```bash
cd ztncui/src  # 进入src目录
npm install
```

在 `src` 目录下创建一个名为 `.env` 的文件，填入以下内容：
```
ZT_TOKEN=#####
NODE_ENV=production
```
其中，`#####` 的值可以通过 `cat /var/lib/zerotier-one/authtoken.secret` 命令获取。

之后复制默认用户：
```bash
cp -v etc/default.passwd etc/passwd
```

此时已完成 ztncui 的基本配置，要让其自动启动，需要依靠 PM2 实现：
```bash
sudo npm install -g pm2  # 安装 PM2
pm2 start bin/www --name ztncui
pm2 startup
pm2 save
```

之后访问 `localhost:3000` 即可。可使用 Nginx 等反向代理实现免端口和 HTTPS 访问。