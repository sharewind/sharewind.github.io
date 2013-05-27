---
layout: post
title: "Java NIO Channel"
date: 2012-12-18 14:46
comments: true
categories: Java NIO
---
《Java NIO》学习笔记 —— Channel篇

####Channel的类层次图

![java channels class](/pics/java-nio-channel-class.jpg)  
[点击查看大图](/pics/java-nio-channel-class.jpg)

channel 主要分为 WritableByteChannel,ReadableByteChannel 以及 NetworkChannel.  
图中有一个 FileChannel类和三个socket通道类：SocketChannel,ServerSocketChannel 和 DatagramChannel.

####Channel通道的接口

	public interface Channel extends Closeable {
	    public boolean isOpen();
	    public void close() throws IOException;
	}

####打开Channel

#####1.socket Channel

socket Channel可以直接被创建

	SocketChannel sc = SocketChannel.open();
	sc.connect(new InetSocketAddress("host",port));

	ServerSocketChannel ssc = ServerSocketChannel.open();
	ssc.socket().bind(new InetSocketAddress(port));

	DatagramChannel dc = DatagramChannel.open();
	
#####2. FileChannel  

FileChannel只能通过一个打开的RandomAccessFile、FileInputStream或FileOutputStream 对象上调用getChannel()方法来获取，不能直接创建一个FileChannel对象。

	RandomAccessFile raf = new RandomAccessFile("filename","r");
	FileChannel fc = raf.getChannel();

**对比**：  
Socket 对象上的getChannel方法: public SocketChannel getChannel(),并不会创建一个channel,当且仅当通过 SocketChannel.open 或 ServerSocketChannel.accept 方法创建了通道本身时，套接字才具有一个通道。

####使用Channel

	public interface ReadableByteChannel extends Channel {
	    public int read(ByteBuffer dst) throws IOException;
	}
	
	public interface WritableByteChannel extends Channel{
	    public int write(ByteBuffer src) throws IOException;
	}
	
	public interface ByteChannel extends ReadableByteChannel, WritableByteChannel{
	
	}

Java的每个file或socket channel 都实现这三个接口，file channel 是否可以读写取决于底层打开文件的方式。  
例如：
	
	FileInputStream is = new FileInputStream(fileName);
	FileChannel fc = is.getChannel();

从输入流获取的FileChannle只能读取，而不能写入。  

Channel 可以以阻塞或非阻塞方式运行。非阻塞的方式永远不会用调用的线程休眠，请求操作要么立即返回，要么返回一个结果表明未进行任何操作。


