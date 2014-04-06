---
layout: post
title: "linux下ssh免输入密码方法"
date: 2013-06-25 04:27
comments: true
categories: 
---

我们在日常开发或者运维过程中需要在不同的主机之间切换或者scp，在内网可信网络环境下重复输入密码是件很繁琐和考验记忆的事情，因此我们可以在自己常用的机器之间通过使用ssh-keygen工具做ssh的授权配置来省去输入密码的过程。

若要实现从A访问B不需要密码直接ssh(使用账户zhangsan)，需要以下步骤：

1.登陆A，并切换到zhangsan: `su - zhangsan`;

2.生成公钥和私钥：`ssh-keygen -t dsa`。其中-t参数后面可以为dsa或rsa，具体类型根据机器环境决定，现在大部分应该是dsa。接下来一路回车后会在~zhangsan/.ssh/目录下生成id_dsa和id_dsa.pub文件，其中id_dsa.pub文件就是公钥文件，需要拷贝到目标及其B上的。

3.拷贝公钥到目标机器: 
`scp id_dsa.pub zhangsan@B:/home/zhangsan/.ssh/id_dsa.pub.A`

4.追加到目标机器~/.ssh/authorized_keys中: `~/.ssh/cat id_dsa.pub.A >> ~/.ssh/authorized_keys`

5.Done
