---
layout: post
title: "java performance diagnosis"
date: 2013-03-05 15:55
comments: true
categories: Java JVM
---

1. top 命令  

	top -H -p pid 列出指定线程中的所有线程，可以查看占用cpu最多的线程。

	jstack pid > thread.dump 从thread dump 中找到占用cpu最高的线程，线程id要转化成16进制。

2. 

	
