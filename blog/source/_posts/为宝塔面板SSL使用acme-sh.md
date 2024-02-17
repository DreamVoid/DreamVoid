---
title: 为宝塔面板SSL使用acme.sh
date: 2022-08-12 01:24:13
tags: 
- 宝塔
- SSL
categories: 技术
---
# 起因

最近宝塔面板的 SSL 续签老是出问题，尤其是有一台机器手动续签 Let's Encrypt 必出现 `KeyError: 'private_key'` 问题，于是想用别的方法进行续签证书。

经过查找，确认宝塔面板的 SSL 存放位置是 `/www/server/panel/ssl`，证书和私钥文件分别是 `certificate.pem` 和 `privateKey.pem`。

所以，只需要通过别的方式自动替换这两个文件就能实现续签证书了。

# 理论存在，实践开始

绝大多数自动化续签 SSL 都是通过 acme.sh 这个工具实现的，所以我也就顺理成章的使用了这个工具。

首先安装 acme.sh，`me@example.com` 是我自己的邮箱：

```bash
curl https://get.acme.sh | sh -s email=me@example.com
```

我的域名使用 Cloudflare 解析，因此使用 DNS 方式申请证书。首次使用前，需要把 Cloudflare 的账户信息加入到环境变量里，我使用的是 API 令牌的方式，执行两条命令：

```bash
export CF_Token="令牌内容"
export CF_Account_ID="账户ID"
```

然后执行命令申请证书：

```bash
acme.sh --issue -d 域名1号 -d 域名 --dns dns_cf
```

实际操作过程中，我在中国大陆的服务器连不上默认的 ZeroSSL CA，因此临时使用 Let's Encrypt 来签发证书，在命令之后加上 `--server letsencrypt` 参数：

```bash
acme.sh --issue -d 域名1号 -d 域名 --dns dns_cf --server letsencrypt
```

等待证书签发，之后就可以执行部署命令了：

```bash
acme.sh --install-cert -d 域名1号 \
--key-file       /www/server/panel/ssl/privateKey.pem  \
--fullchain-file /www/server/panel/ssl/certificate.pem \
--reloadcmd     "bt 1"
```

执行完后，证书就部署好了，刷新网页也能看到新申请的证书了。

![新证书申请成功后](/image/新证书申请成功后.png)

根据 acme.sh 所述，证书到期前 30 天会自动续签，就看是否真的能续签成功了。