---
layout:     post   				    # 使用的布局（不需要改）
title:      2019-5-20-在Centos上复习Linux命令			# 标题 
subtitle:   常见Linux命令 #副标题
date:       2019-5-20 				# 时间
author:     Tempo					# 作者
header-img: img/post-bg-keybord.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Centos
    - Linux
    - bash
---
# 在Centos上复习Linux命令

#### 进程端口类
* 查看进程：netstat -anp|grep 80 
* 杀死进程：kill xxxx(加急版，kill -9)
* 后台调用进程：nohup ./target/appname-version &
* 查看后台调用进程log：tail -f nohup.out
#### 防火墙
* 开启：systemctl start firewalld
* 关闭：systemctl stop firewalld
* 设置开机启动：systemctl enable firewalld
* 停止并禁止开机启动：systemctl disable firewalld
* 查看状态 systemctl status firewalld