---
title: 使用Nginx搭建Minecraft加速IP
date: 2020-05-01 15:32:45
tags: 
- Nginx
- 加速IP
- Minecraft
categories: 游戏
---
大致就是用Nginx建一个Stream配置文件实现流量转发

B站视频版：https://www.bilibili.com/video/BV1jz411z77Y
# 准备
首先，需要在bt.cn获取宝塔面板，这里就不作说明了

我用的测试服务器是Ubuntu，所以使用相应的指令
``` bash
$ wget -O install.sh http://123.129.198.197/install/install-ubuntu_6.0.sh && sudo bash install.sh'
```
然后等安装完成后进入后台使用用户名和密码登录

打开侧栏的“软件商店”，选择Nginx服务安装，然后就是漫长的等待

# 配置
Nginx安装完成后，进入配置文件界面，删掉原来的HTTP部分，把以下文件的代码复制到末尾
`http://dl.dreamvoid.ml/file/17787109-440778997`

然后点击重载配置，一个加速IP就搭建好了

# 连接
以Hypixel为例，Hypixel现在不允许直接使用数字IP进入服务器，所以需要依靠域名，但是Hypixel并没有限制是什么域名，因此我们可以使用自己的域名
## 使用hosts文件免域名连接
打开以下路径

`C:\Windows\System32\drivers\etc`

找到hosts文件，使用记事本或者Notepad++（如果有安装）打开

把以下内容复制到文件末尾：

`服务器IP 域名`

比如：

`139.199.223.159 hypixel.gg`

这就把“hypixel.gg”自动解析到IP“139.199.223.159”，因为hosts文件相当于本地DNS服务器

然后进入游戏输入hypixel.gg即可进入服务器游玩
## 使用域名解析
这里我不多描述，因为百度关于域名解析的教程一大堆

可以去阿里云、腾讯云之类的域名提供商买域名

如果没钱的话，可以用 https://freenom.com/ 注册免费域名，都是一样的
# 视频
应该能看吧
<iframe src="//player.bilibili.com/player.html?aid=200445265&amp;bvid=BV1jz411z77Y&amp;cid=185486449&amp;page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" __idm_id__="116570113" height="600" width="100%"> </iframe>