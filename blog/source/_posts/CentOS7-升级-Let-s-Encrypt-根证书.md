---
title: CentOS7 升级 Let's Encrypt 根证书
date: 2021-10-04 00:36:32
tags: SSL
categories: 技术
---
Lets Encrypt 根证书于 2021 年 9 月 30 日到期。CentOS 7 上都会受到影响，升级CA证书即可修复。

相关Ⅰ：https://letsencrypt.org/docs/dst-root-ca-x3-expiration-september-2021/

相关Ⅱ：https://access.redhat.com/errata/RHBA-2021:3649

快速修复：```rpm -qa | grep ca-certificates-2021.2.50-72.el7_9.noarch || yum update -y  ca-certificates || update-ca-trust```