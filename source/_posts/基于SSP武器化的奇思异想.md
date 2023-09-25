---
title: 基于SSP武器化的奇思异想
date: 2020-06-15 11:51:24
description: 极客兔兔的小站，致力于分享一些技术教程和有趣的技术实践。

---

# 前言

Windows Server 2008 R2及更高版本的系统默认无法获得明文密码，但是可以利用SSP即 Security Support Provider,通俗理解就是一个用于身份验证的 **dll**,系统在启动时 SSP 会被加载到 lsass.exe 进程中,由于 lsa 可扩展,导致在系统启动时我们完全可以加载一个自定义的 dll来记录明文账号密码。

mimilib从lsass进程中提取明文凭据的实现代码：

https://github.com/gentilkiwi/mimikatz/blob/master/mimilib/kssp.c

自动武器化实现思路：

1、将动态库或是驱动文件打包进一个可执行文件中，再由需要使用的时候，再临时释放和加载。

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/1610437864349-e8b1e547-3ef1-4817-9279-19d0ccf64fec.png)

2、将释放的dll文件复制到c:\windows\system32下，这里编译的时候需要注意的是程序是32位的，在64位系统下，所有对system32的操作都会被转向syswow64，修改系统注册表Security Packages的值为要加载dll的名称。

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/1610437864244-b52bc924-d010-4b63-a419-63fe39d86cee.png)

3、通过调用 AddSecurityPackage API函数可以使 lsass.exe 进程加载指定的SSP / AP，无需重启，此时管理员输入了新的凭据，将会生成文件kiwissp.log，实战中,可配合HTTP ServerAPI端口复用进行实时获取凭证，如有需求也可以自行改进。

效果演示：

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/1610437864356-32c8ce81-c89a-4a58-8ef3-9118ffb93f7d.png)

# Reference

http://www.mamicode.com/info-detail-2638519.html

https://www.4hou.com/posts/y6jW

https://blog.csdn.net/ayang1986/article/details/83351842