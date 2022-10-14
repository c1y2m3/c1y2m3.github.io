---
title: 探索基于HTTP.SYS实现权限维持
date: 2019-11-21 11:51:24
description: 极客兔兔的小站，致力于分享一些技术教程和有趣的技术实践。




---

 在通常WEB应用中，一个应用只能绑定一个端口，若同时绑定一个端口，必然会抛出端口占用异常信息，微软在IIS 5.0以后版本提供了一种机制，Net. Port Sharing,即端口共享。它允许对应用同时使用一个端口只需要各自注册的url前缀不一样即可，

这种机制被封装在http.sys驱动中，驱动监听HTTP流量，然后根据URL注册的情况去分发到当前URL对应的应用。

# Winrm服务端口复用

实际上，winrm服务就是基于http.sys驱动上注册了wsman后缀，Windows Server 2008默认关闭WinRM服务，Windows Server 2012默认开启

测试环境：192.168.1.201 系统版本：Windows Server R2

在命令行下开启服务，防火墙会自动添加例外规则，系统默认监听端口5985。

winrm quickconfig –q

![img](https://www.yunzhijia.com/microblog/filesvr/5dd643c2b54c8d62368ed46e)

此时使用netsh http show servicestate

可以查看到http.sys新注册了一条url前缀：

![img](https://www.yunzhijia.com/microblog/filesvr/5dd643e06d67ff57e939b1d0)

由于系统原本没有开启5985端口，为了增加后门的隐蔽性，故通过以下命令将winrm服务端口修改至80端口，达到端口复用的效果。

winrm set winrm/config/Listener?Address=*+Transport=HTTP @{Port=”80”}

![img](https://www.yunzhijia.com/microblog/filesvr/5dd6440290144e0b52ff4fd5)

默认客户端连接则会提示Winrs error:WinRM 客户端无法处理该请求。如果身份验证方案与 Kerberos 不同，或者客户端计算机…,因为WinRM只允许当前域用户或者处于本机TrustedHosts列表中的远程主机进行访问，则需在客户端添加一个TrustedHost表，*代表信任远程任意主机：

Set-Item WSMan:localhost\client\trustedhosts -value *

然后使用winrs命令连接远程web服务端口获得交互式SHELL,

![img](https://www.yunzhijia.com/microblog/filesvr/5dd64426a3725915003627e4)

使用python实现一款支持NTLM hash的客户端，需要注意的是：

windows 2008是LM-HASH:NTLM-HASH的方式，使用32个0替代。
windows 2012以及之后只能抓到NTLM的Hash，直接使用即可。

最终实现的代码已上传至github，地址如下：

https://github.com/c1y2m3/python-tools/blob/master/WinrmAttack.py
![img](https://www.yunzhijia.com/microblog/filesvr/5ddbefaeb54c8d62369991a6)

# HTTP ServerAPI端口复用

 微软对外开放了如何调用这种http.sys驱动机制的API接口，也就是HTTP Server API。HTTP Server API 运行在用户模式中，也就是说任意用户都可以调用该API实现一个HttpListener，与 IIS 共享端口，但是前提是你必须拥有管理员权限。

![img](https://www.yunzhijia.com/microblog/filesvr/5dd64458b54c8d62368ed80e)

上图整个过程描述如下：

（1）第一步，IIS 或者是自己写的程序（HttpListener）调用API ，向内核的Http.sys注册一个URL前缀，这相当于向路由器添加一条路由规则，Http.sys就是这个路由器。

（2）第二步，Http.sys捕获到一个http请求，它将会根据自身的“路由表”找到该http请求的前缀所对应的应用，然后把请求分发给该应用。

（3）第三步，HttpListener接收到Http.sys转发来的请求，对其进行回应。

# 1.示例代码测试

微软官方有一个基于HTTP Server API 1.0版本的demo：

https://docs.microsoft.com/zh-cn/windows/win32/http/http-server-sample-application

笔者对其demo进行了简单的修改，进行如下演示：

![img](https://www.yunzhijia.com/microblog/filesvr/5dd6448e28011f0fc49af9f1)

通过pReq->CookedUrl.pQueryString 能够读取url中的参数，去除参数后的问号对字符串拼接，

获取客户端输入的内容，C++ 代码如下：

```
*QueryString = new WCHAR[pReq->CookedUrl.QueryStringLength - 1];

wcsncpy_s(QueryString, wcslen(QueryString), pReq->CookedUrl.pQueryString + 5, wcslen(QueryString) - 1);

wprintf(L"[*] QueryStringDecode:%ws\n", QueryString);
```

![img](https://www.yunzhijia.com/microblog/filesvr/5dd644b36d67ff57e939b70f)

# 后门功能设计

基于该demo进行修改，设计命令执行后门，通过GET请求发送要执行的cmd命令，格式为?，对传来的http请求进行身份验证，验证成功时进行回应，

例如: http://127.0.0.1/a?png=whoami，要执行的命令为whoami，通过处理http请求内容，执行系统命令并通过SendHttpResponse回显到客户端页面。

使用管道读取命令并执行，回传结果，参考代码如下：

https://github.com/3gstudent/Homework-of-C-Language/blob/master/UsePipeToExeCmd.cpp

![img](https://www.yunzhijia.com/microblog/filesvr/5dd644fc32f2ca155fc9e81b)

![img](https://www.yunzhijia.com/microblog/filesvr/5dd64511b54c8d62368edc4b)

在测试过程中发现当传入特殊字符时，例如空格，单双，引号等字符会被编码，导致某些命令无法解析执行 ,如图所示:

![img](https://www.yunzhijia.com/microblog/filesvr/5dd64543a372591500362ea7)

这里笔者对命令使用了base64解码处理，解决以上问题且可以对流量进行简单加密，以及对回传的结果作格式转换，\n换行符转换成html中的
。

并使用#pragma comment( linker, “/subsystem:\”windows\” /entry:\”wmainCRTStartup\”” )，屏蔽控制台应用程序的窗口，修改测试代码如下图：

![img](https://www.yunzhijia.com/microblog/filesvr/5dd645646d67ff57e939bb55)

# 效果演示

最终实现的代码已上传至github，地址如下：

https://github.com/c1y2m3/python-tools/blob/master/IIS_backdoor/main.cpp

可以看到，未注册/a/前缀时，因为此时只有iis使用了端口共享机制，http.sys把访问/a/的请求交给了iis处理，iis返回了404页面。

![img](https://www.yunzhijia.com/microblog/filesvr/5dd6459532f2ca155fc9eb78)

随后我们用自己的程序把/a/前缀注册到http.sys，此时访问/a/路径,http.sys于是把请求交了我们的后门程序并回显系统命令。

![img](https://www.yunzhijia.com/microblog/filesvr/5dd645b16d67ff57e939bd03)