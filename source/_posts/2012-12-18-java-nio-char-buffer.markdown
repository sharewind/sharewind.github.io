---
layout: post
title: "Java NIO Char Buffer"
date: 2012-12-18 11:47
comments: true
categories: Java NIO
---

《Java NIO》学习笔记 —— Char Buffer篇

####字节顺序

java.nio.ByteOrder 字节顺序的类型安全枚举。  

- BIG_ENDIAN 表示 big-endian 字节顺序的常量。
- LITTLE_ENDIAN 表示 little-endian 字节顺序的常量。
- nativeOrder() 获取底层平台的本机字节顺序。

IP协议规定了使用大端的网络字节顺序。  

除ByteBuffer外的 Buffer通过创建或包装数组元素产生的, 其order方法返回的字节序与 ByteOrder.nativeOrder()返回的值相同;  
如果作为ByteBuffer 的视图缓冲区而创建的Buffer，缓冲区的字节顺序是创建视图时ByteBuffer缓冲区的字节顺序。
  
ByteBuffer 默认字节顺序总是 BIG_ENDIAN，无论系统固有的字节顺序是什么。ByteBuffer可以通过order(ByteOrder bo)方法来改变字节顺序。