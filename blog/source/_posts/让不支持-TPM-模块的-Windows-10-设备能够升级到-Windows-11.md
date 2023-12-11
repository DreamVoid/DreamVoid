---
title: 让不支持 TPM 模块的 Windows 10 设备能够升级到 Windows 11
date: 2021-10-20 00:33:22
tags: Windows
categories: 技术
---
微软发布了 Windows 11，我自然是想尝试一下，但怎奈 Windows 11 又要 TPM 又要安全启动，我的老主板没法支持。

哪里有压迫，哪里就有反抗。我找了一下资料，发现了绕过这两个限制的方法。（[参考资料](https://www.seozhh.com/12628.html)）

1. 打开“注册表编辑器”（Win + R，键入```regedit```）；
2. 定位到“```计算机\HKEY_LOCAL_MACHINE\SYSTEM\Setup```”，在此处新建“项”，名称为 ```LabConfig```；
3. 新建两个 DWORD 值，键名分别为 ```BypassTPMCheck``` 和 ```BypassSecureBootCheck```，十六进制值为 ```00000001```
4. 此时已绕过两个检测。

虽然不知道这样搞未来的支持会怎样，但先体验了再说。