---
title: 自建 ZeroTier 组网
date: 2024-10-16 21:22:25
tags: ZeroTier
categories: 技术
---
ZeroTier 是一个好的组网软件，搭配上自建的 planet 根服务器能让体验更上一层楼。

# 搭建 planet 根节点
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
## 使用源码搭建
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

## 使用 deb 包搭建（2026 年 3 月 4 日更新）
发现 ztncui 是有官方 deb 安装包的，使用 deb 包可以方便的管理软件，也省去直接使用源码的麻烦。（[官方安装说明](https://key-networks.com/ztncui/#installation)）

首先下载 deb 包：
```bash
curl -O https://s3-us-west-1.amazonaws.com/key-networks/deb/ztncui/1/x86_64/ztncui_0.8.14_amd64.deb
```

安装：
```bash
apt install ./ztncui_0.8.14_amd64.deb
```

将 ZeroTier 访问密钥写入 ztncui：
```bash
sh -c "echo ZT_TOKEN=`cat /var/lib/zerotier-one/authtoken.secret` > /opt/key-networks/ztncui/.env"
```

设置其他的配置：
```bash
sh -c "echo NODE_ENV=production >> /opt/key-networks/ztncui/.env"
chmod 400 /opt/key-networks/ztncui/.env
chown ztncui.ztncui /opt/key-networks/ztncui/.env
```

重启服务：
```bash
systemctl restart ztncui
```

最后就可以在 `localhost:3000` 看到面板了。默认用户名 `admin`，密码 `password`。

# 搭建 moon 中继节点
moon 除了能够作为中继节点以外，如果有两台服务器的内网互相是通的，那么可以通过制作一个指向内网 IP 的 moon 文件让这两台机器通过内网而不是公网连接。

进入 ZeroTier 目录：
```bash
cd /var/lib/zerotier-one
```

生成 `moon.json`：
```bash
zerotier-idtool initmoon identity.public >> moon.json
```

编辑 `moon.json`，将 IP 地址信息写入 `stableEndpoints`，像下面这样：
```json
{
    "id": "**********",
    "objtype": "world",
    "roots": [
        {
            "identity": "**********",
            "stableEndpoints": ["10.0.1.1/9993", "fe80::1/9993"]
        }
    ],
    "signingKey": "**********",
    "signingKey_SECRET": "**********",
    "updatesMustBeSignedBy": "**********",
    "worldType": "moon"
}
```

生成 moon：
```bash
zerotier-idtool genmoon moon.json
```

之后会得到一个 moon 拓展名的文件，这就是节点文件了。

如果没有 `moons.d` 文件夹，先新建一个：
```bash
mkdir moons.d
```

如果是面向公网的 moon，可以将 moon 文件移动进去 `mv 000000**********.moon moons.d/`，这样其他设备就可以通过 `zerotier-cli orbit` 命令直接下载。如果是面向内网的 moon，那么就手动下载文件放到其他节点的 `moons.d` 文件夹里。

重启 ZeroTier 服务：
```bash
systemctl restart zerotier-one
```