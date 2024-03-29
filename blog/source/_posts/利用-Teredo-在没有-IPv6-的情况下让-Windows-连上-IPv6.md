---
title: 利用 Teredo 在没有 IPv6 的情况下让 Windows 连上 IPv6
date: 2021-10-17 18:00:42
tags: IPv6
categories: 技术
---
参考资料：https://www.jianshu.com/p/a821a874fbfc

---

测试过能连上 IPv6 的服务。

1. 按下键盘上的 ```Win + R```，然后输入 ```gpedit.msc``` 回车打开组策略编辑器。
2. 依次打开```计算机配置```——```管理模板```——```网络```——```TCPIP 设置```——```IPv6 转换技术```
3. （可选）“**6to4 状态**”和“**ISATAP 状态**”设置为“**已启用**”，然后在下方的配置窗口中选择“**已禁用状态**”。
4. “**Teredo 状态**”设为“**企业客户端**”；“**Teredo 默认限定**”设为“**已启用状态**”
5. “**Teredo 服务器名称**” 设为“```teredo.remlab.net```”（我用的是这个）

自此，如果没有任何问题，IPv6 就已经连上了。

可以在 http://test-ipv6.com/index.html.zh_CN 测试是否连上 IPv6.
