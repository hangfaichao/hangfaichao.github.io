---
title: MacOS根目录创建文件/文件夹失败“Read-only file system”
date: 2021-07-21 14:56:44
tags:
category: OS
---

公司项目运行过程有根目录的文件创建编辑权限，一些配置信息以及log需要写在根目录上。而Mac在根目录创建文件时提示“Read-only file system”。

系统：Big Sur

以下方式都不可用：

1、sudo创建

```shell
sudo touch test
```

2、进入恢复模式->关闭SIP->重新挂载到根目录

（1）重启电脑按住cmd+R进入恢复模式

（2）从右上角找到终端，输入命令`csrutil disable`关闭SIP

（3）重启之后输入命令`sudo mount -uw /`重新挂载

（4）重新进入恢复模式开启SIP`csrutil enable`

---

最终尝试如下方法生效：

1、在用户目录创建需要在根目录创建的文件夹，这里以"app"和"data"为例，创建在用户目录"/Users/zhh/"下面

2、在`\etc\synthetic.conf`文件（如果没有则创建）中输入以下内容（注意中间是tab键：

```
app		/Users/zhh/app
data	/Users/zhh/data
```

3、重新启动电脑就会发现根目录下多了"app"和"data"两个文件夹，可以发现系统帮我们将这两个文件夹软链接到了用户目录上面对应的文件夹了。

![image-20210721151947540](image-20210721151947540.png)



