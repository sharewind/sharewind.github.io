---
layout: post
title: "Linux服务器时间同步"
date: 2013-03-19 16:23
comments: true
categories: Linux Shell
---

经常有各种因为服务器时间不一致引发的各种血案：  

1. 依赖系统时间生成的ID序列  
2. 依赖系统时间进行同步的协议  
3. 插入的数据依赖于数据库服务器的时间

同时查看多服务器时间的脚本  

	for ip in {10.1.1.10,10.1.1.12};do ssh root@$ip date;done

更新服务器时间

	/usr/sbin/ntpdate ntp.sohu.com > /dev/null 2>&1;/sbin/clock -w
