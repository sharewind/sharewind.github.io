---
layout: post
title: "Java NIO Buffer"
date: 2012-12-12 10:38
comments: true
categories: Java NIO
---
《Java NIO》学习笔记 —— Buffer篇

#### Java Buffer 类图
![java buffer class](/pics/java-buffer-class.jpg)

#### 缓存区的基本概念

**容量(Capacity)**  
缓冲区能够容纳的数据元素的最大数量。这一容量在缓冲区创建时被设定，并且永远不能被改变。

**上界(Limit)**  
缓冲区的第一个不能被读或写的元素。或者说，缓冲区现存元素的计数。

**位置（Position）**  
下一个要被读或写的元素的索引。位置会自动由相应的get()和put() 方法更新。

**标志（Mark）**   
一个标志位置。调用mark()来设定mark = position。调用reset() 设定position = mark。标记在设定之前是未定义的(undefined)。


####Buffer 的主要方法

	public abstract class Buffer {
	 
	    Buffer(int mark, int pos, int lim, int cap)
	 
	    public final int capacity() 
	  
	    public final int position()     
	    public final Buffer position(int newPosition) 
	   
	    public final int limit()      
	    public final Buffer limit(int newLimit)       
	 
	    public final Buffer mark()     
	    public final Buffer reset() 
	    
	    public final Buffer clear()
	
	    public final Buffer flip()
	    public final Buffer rewind() 
	    
	    public final int remaining()
	    public final boolean hasRemaining() 
	   
	    public abstract boolean isReadOnly();
	    public abstract boolean isDirect();
	}

ByteBuffer 的主要方法

	public abstract class ByteBuffer
	    extends Buffer
	    implements Comparable<ByteBuffer>{
		public abstract byte get();
		public abstract byte get(int index);
		public abstract ByteBuffer put(byte b);
		public abstract ByteBuffer put(int index, byte b);
	}

**1. 填充**  
调用put方法把byte 放入Buffer，初始状态position=0,limit = capacity;  
buffer.put((byte)'H').put((byte)'e').put((byte)'l').put((byte)'l').put((byte)'o');

**2. 翻转**  
当写入完成后，我们需要设定上限来指定写入的有效内容的末端，然后将位置重置为0.  
buffer.limit(buffer.position()).position(0)

可以用buffer.flip(); 方法将缓存区从可以继续写入的状态 翻转成一个准备读出的状态。  

**rewind()** 方法与flip()类似，但不影响上界属性，它只是将位置值设回0，可以重新读取已被翻转的缓冲区的数据。  
如果将缓冲区翻转两次，会将实际大小变为0，position和limit 都为0。

**3. 释放**(即读取数据)

	int count = buffer.remaining();
	byte[] bytes = new byte[cout]; 
	int i = 0;
	while(buffer.hasRemaining()){
		bytes[i++] = buffer.get();
	}

**4. 压缩**  
有时候只想从缓冲区释放一部分数据，而不是全部，然后重新填充。API为我们提供了compact 方法，compact 方法将会复制当前position未读取的数据到缓存区开头，然后重新设置 position = limit - position, 并将limit 恢复到capacity, limit = capacity。  
应用场景：  

	public class ChannelCopy {
	
		public static void main(String[] args) throws IOException {
			ReadableByteChannel in = Channels.newChannel(System.in);
			WritableByteChannel out = Channels.newChannel(System.out);
			copyChannel(in,out);
			System.out.println("copy1");
	//		copyChannel2(in,out);
	//		System.out.println("copy2");
			in.close();
			out.close();
		}
	
		private static void copyChannel2(ReadableByteChannel src, WritableByteChannel dest) throws IOException {
	
			ByteBuffer buffer = ByteBuffer.allocate(1024);
			while(src.read(buffer) > 0 ){
				buffer.flip();
				dest.write(buffer);
				buffer.compact();
			}
	
			buffer.flip();
			while(buffer.hasRemaining()){
				dest.write(buffer);
			}
	
		}
	
		private static void copyChannel(ReadableByteChannel src, WritableByteChannel dest) throws IOException {
	
			ByteBuffer buffer = ByteBuffer.allocate(1024);
			while(src.read(buffer) > 0 ){
				buffer.flip();
				while(buffer.hasRemaining()){
					dest.write(buffer);
				}
				buffer.clear();
			}
	
		}
	
	}

**5. 比较**

	public abstract class ByteBuffer implements Comparable<ByteBuffer>{
		public boolean equals(Object obj)
		public int compareTo(Object obj)
	}

两个缓冲区被认为相等的充要条件：  

- 两个对象类型相同。
- 两个对象都剩余同样数量的元素
- 在每个缓冲区中应被get() 方法返回的剩余数据元素序列必须一致。

ByteBuffer.equals 源码

    public boolean equals(Object ob) {
        if (this == ob)
            return true;
        if (!(ob instanceof ByteBuffer))
            return false;
        ByteBuffer that = (ByteBuffer)ob;
        if (this.remaining() != that.remaining())
            return false;
        int p = this.position();
        for (int i = this.limit() - 1, j = that.limit() - 1; i >= p; i--, j--)
            if (!equals(this.get(i), that.get(j)))
                return false;
        return true;
    }

**6. 批量移动**

	public abstract class ByteBuffer extends Buffer implements Comparable<ByteBuffer>{
		public ByteBuffer get(byte[] dst, int offset, int length)  
		public ByteBuffer get(byte[] dst)  
		public ByteBuffer put(ByteBuffer src)  
		public ByteBuffer put(byte[] src, int offset, int length)  
		public final ByteBuffer put(byte[] src)
	}

● 从buffer中读取数据到数组  
ByteBuffer源码：    

    public ByteBuffer get(byte[] dst, int offset, int length) {
        checkBounds(offset, length, dst.length);
        if (length > remaining())
            throw new BufferUnderflowException();
        int end = offset + length;
        for (int i = offset; i < end; i++)
            dst[i] = get();
        return this;
    }

 
    public ByteBuffer get(byte[] dst) {
        return get(dst, 0, dst.length);
    }

 ● 把另外一个buffer剩余的数据放入Buffer中  
ByteBuffer源码：  
 
    public ByteBuffer put(ByteBuffer src) {
        if (src == this)
            throw new IllegalArgumentException();
        int n = src.remaining();
        if (n > remaining())
            throw new BufferOverflowException();
        for (int i = 0; i < n; i++)
            put(src.get());
        return this;
    }

 ● 把数组元素放入Buffer中  
ByteBuffer源码：  

    public ByteBuffer put(byte[] src, int offset, int length) {
        checkBounds(offset, length, src.length);
        if (length > remaining())
            throw new BufferOverflowException();
        int end = offset + length;
        for (int i = offset; i < end; i++)
            this.put(src[i]);
        return this;
    }
 
    public final ByteBuffer put(byte[] src) {
        return put(src, 0, src.length);
    }

**7. 线程安全**  
多个当前线程使用缓冲区是不安全的。如果一个缓冲区由不止一个线程使用，则应该通过适当的同步来控制对该缓冲区的访问。 


####参考资料

- [java nio Buffer 中 compact的作用](http://blog.csdn.net/jiang_bing/article/details/7878390)






