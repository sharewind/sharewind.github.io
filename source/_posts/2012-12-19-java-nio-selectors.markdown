---
layout: post
title: "Java NIO Selectors"
date: 2012-12-19 00:20
comments: true
categories: Java NIO
---
#### 一、Selector 基础

**选择器（Selector）**  
选择器管理着一个被注册通道集合的信息和它们的就绪状态。

**可选择的通道（SelectableChannel）**
这个抽象类提供了实现通道可选择性所需要的公共方法。它是所有支持就绪检查的通道类的父类。FileChannel是不可选择的，所有的socket通道都是可以选择的，包括从pipe对象中获取的。

**选择键(SelectionKey)**  
选择键封装了特定的通道与特定的选择器的注册关系。通过调用 SelectableChannel.register() 注册方法会返回选择键。选择键包含了两个属性： interestOps 指示注册关系所关心的通过操作，readyOps 指示通道已经准备好的操作。

####选择器相关的类层次图
![](/pics/java-nio-selectors-class.jpg)  
[点击查看大图](/pics/java-nio-selectors-class.jpg)

**SelectableChannel**相关的API方法： 

	public abstract class SelectableChannel extends AbstractInterruptibleChannel implements Channel{
	     
	    public final SelectionKey register(Selector sel, int ops)
		public abstract SelectionKey register(Selector sel, int ops, Object att) throws ClosedChannelException;
	
	    public abstract boolean isRegistered();   
	    public abstract SelectionKey keyFor(Selector sel); 
	
	    public abstract SelectableChannel configureBlocking(boolean block)  throws IOException;
	    public abstract boolean isBlocking();
	    public abstract Object blockingLock();
	    
		public abstract int validOps(); 
	}

- 在通道注册到选择器之前必须配置为非阻塞模式(通过调用 configureBlocking(false))  
- 通过调用validOps( )方法来获取特定的通道所支持的操作集合。  
- 一个单独的通道对象可以被注册到多个选择器上。可以调用isRegistered( )方法来检查一个通道是否被注册到任何一个选择器上。一个SelectionKey被取消后，并不会马上被注销（可能有时间延迟），所以isRegistered（）并不是确切答案。

**Selector**相关的API方法： 

	public abstract class Selector implements Closeable {
	 
	    public static Selector open() throws IOException ;
	    public abstract boolean isOpen();
	 
	    public abstract Set<SelectionKey> keys(); 
	    public abstract Set<SelectionKey> selectedKeys();
	
	 
	    public abstract int selectNow() throws IOException; 
	    public abstract int select(long timeout)throws IOException; 
	    public abstract int select() throws IOException;
	 
	    public abstract Selector wakeup(); 
	    public abstract void close() throws IOException;
	}


SelectableChannel 尽管也定义了register方法，但最终都是将Channel注册到 Selector 上。Selector 维护了一个需要监控的Channel 集合。register 方法会返回一个建立了两个对象关联的SelctionKey。一个给定的Channel可以被注册到多个Selector上，而且不需要知道它被注册到哪个Selector上。

**AbstractSelectableChannel**中register方法的实现:  

	    public final SelectionKey register(Selector sel, int ops, Object att) throws ClosedChannelException {
	        if (!isOpen())
	            throw new ClosedChannelException();
	        if ((ops & ~validOps()) != 0)
	            throw new IllegalArgumentException();
	        synchronized (regLock) {
	            if (blocking)
	                throw new IllegalBlockingModeException();
				
				// 查找已注册到selector 上的SelectionKey,如果有则对原有的Key进行更新
	            SelectionKey k = findKey(sel);
	            if (k != null) {
	                k.interestOps(ops);
	                k.attach(att);
	            }
	            if (k == null) {
	                // New registration // 调用了AbstractSelector的register方法
	                k = ((AbstractSelector)sel).register(this, ops, att);
	                addKey(k);
	            }
	            return k;
	        }
	    }

**AbstractSelector**的register方法定义

    protected abstract SelectionKey register(AbstractSelectableChannel ch,
                                             int ops, Object att);


**SelectionKey**的主要API：  

	public abstract class SelectionKey {
		
		public static final int OP_READ = 1 << 0;
		public static final int OP_WRITE = 1 << 2;
		public static final int OP_CONNECT = 1 << 3;
		public static final int OP_ACCEPT = 1 << 4;
	
	    public abstract SelectableChannel channel();
	    public abstract Selector selector();
	
	    public abstract boolean isValid();
	    public abstract void cancel();
	
	    public abstract int interestOps();
	    public abstract SelectionKey interestOps(int ops);
	    public abstract int readyOps();
	
	    public final boolean isReadable(){
	    	return (readyOps() & OP_READ) != 0;
	    }
	    public final boolean isWritable()
	    public final boolean isConnectable()
	    public final boolean isAcceptable()
	
	    public final Object attach(Object ob)
	    public final Object attachment()
	}

**SelectionKey**
每次向选择器注册通道时就会创建一个选择键。通过调用某个键的 cancel 方法、关闭其通道，或者通过关闭其选择器来取消 该键之前，它一直保持有效。取消某个键不会立即从其选择器中移除它；相反，会将该键添加到选择器的已取消键集，以便在下一次进行选择操作时移除它。可通过调用某个键的 isValid 方法来测试其有效性。 


#####1. **AbstractSelectionKey**中**cancel()**方法的实现 

	    public final void cancel() {
	        // Synchronizing "this" to prevent this key from getting canceled
	        // multiple times by different threads, which might cause race
	        // condition between selector's select() and channel's close().
	        synchronized (this) {
	            if (valid) {
	                valid = false;
	                ((AbstractSelector)selector()).cancel(this);
	            }
	        }
	    }

#####2. **AbstractSelector**中**cancel()**方法的实现 

	    void cancel(SelectionKey k) {                       // package-private
	        synchronized (cancelledKeys) {
	            cancelledKeys.add(k);
	        }
	    }

#####3. **SelectorImpl**中的select方法

	    public int select(long timeout)throws IOException{
	        if (timeout < 0)
	            throw new IllegalArgumentException("Negative timeout");
	        return lockAndDoSelect((timeout == 0) ? -1 : timeout);
	    }
	
	    public int select() throws IOException {
	        return select(0);
	    }
	
	    private int lockAndDoSelect(long timeout) throws IOException {
	        synchronized (this) {
	            if (!isOpen())
	                throw new ClosedSelectorException();
	            synchronized (publicKeys) {
	                synchronized (publicSelectedKeys) {
	                    // 调用 doSelect 抽象方法
						return doSelect(timeout);
	                }
	            }
	        }
	    }

#####4. Windows平台Selector的实现类**WindowsSelectorImpl**中的doSelect方法

	    protected int doSelect(long timeout) throws IOException {
	        if (channelArray == null)
	            throw new ClosedSelectorException();
	        this.timeout = timeout; // set selector timeout
	
			// 处理取消注册队列,位于 SelectorImpl.processDeregisterQueue()
	        processDeregisterQueue();
	        if (interruptTriggered) {
	            resetWakeupSocket();
	            return 0;
	        }
			.... 
			processDeregisterQueue();// select结束时，再执行一次取消注册的方法
			
			

#####5.**SelectorImpl.processDeregisterQueue()**

    void processDeregisterQueue() throws IOException {
        // Precondition: Synchronized on this, keys, and selectedKeys
        Set cks = cancelledKeys();
		// 获取所有取消的selectionKey，遍历进行移除操作
        synchronized (cks) {
            if (!cks.isEmpty()) {
                Iterator i = cks.iterator();
                while (i.hasNext()) {
                    SelectionKeyImpl ski = (SelectionKeyImpl)i.next();
                    try {
                        implDereg(ski);
                    } catch (SocketException se) {
                        IOException ioe = new IOException(
                            "Error deregistering key");
                        ioe.initCause(se);
                        throw ioe;
                    } finally {
                        i.remove();
                    }
                }
            }
        }
    }



**建立选择器**

	Selector selector = Selector.open();
	channel1.register(selector,SelectionKey.OP_READ);
	channel2.register(selector,SelectionKey.OP_WRITE);
	channel3.register(selector,SelectionKey.OP_READ| OP_WRITE);
	readyCount = selector.select(10000);

select()方法将线程置于休眠状态，直到感兴趣的操作中一个发生或10秒钟时间过去。  
Selector中的open方法，通过SelectorProvider.openSelector()来创建Selector实例：

	public static Selector open() throws IOException {
        return SelectorProvider.provider().openSelector();
    }

当不再需要Selector 时，可以调用 close()方法来释放它可能占用的资源，并将所有相关的SelectionKey设置为无效。

**SelectionKey**

SelectionKey表示注册关系，可以调用 cancel方法取消注册；也可以调用 isValid()来判断是否有效。  
当键被取消时，它将被放在相关的选择器的已取消的键的集合里。注册不会立即被取消，但键会立即失效（参见4.3节）。当再次调用select( )方法时（或者一个正在进行的select()调用结束时），已取消的键的集合中的被取消的键将被清理掉，并且相应的注销也将完成。

当通道关闭时，所有相关的键会自动取消（记住，一个通道可以被注册到多个选择器上）。当选择器关闭时，所有被注册到该选择器的通道都将被注销，并且相关的键将立即被无效化（取消）。一旦键被无效化，调用它的与选择相关的方法就将抛出CancelledKeyException。

可以通过调用键的readyOps( )方法来获取相关的通道的已经就绪的操作。ready集合是interest集合的子集，并且表示了interest集合中从上次调用select( )以来已经就绪的那些操作。

	if((key.readyOps() & SelectionKey.OP_READ) != 0){
		buffer.clear();
		key.channel.ready(buffer);
		buffer.flip();
		// do sth..
	}

需要注意的是，通过相关的选择键的readyOps( )方法返回的就绪状态指示只是一个提示，不是保证。

#### 二、使用Selector

**三种选择键集**  
通过 SelectionKey 对象来表示可选择通道到选择器的注册。选择器维护了三种选择键集：   
● **键集** 包含的键表示当前通道到此选择器的注册。此集合由 keys 方法返回。  
● **已选择键集** 是这样一种键的集合，即在前一次选择操作期间，检测每个键的通道是否已经至少为该键的相关操作集所标识的一个操作准备就绪。此集合由 selectedKeys 方法返回。已选择键集始终是键集的一个子集。   
● **已取消键集** 是已被取消但其通道尚未注销的键的集合。不可直接访问此集合。已取消键集始终是键集的一个子集。 

在新创建的选择器中，这三个集合都是空集合。 

通过某个通道的 register 方法注册该通道时，所带来的副作用是向选择器的键集中添加了一个键。在选择操作期间从键集中移除已取消的键。键集本身是不可直接修改的。 

不管是通过关闭某个键的通道还是调用该键的 cancel 方法来取消键，该键都被添加到其选择器的已取消键集中。取消某个键会导致在下一次选择操作期间注销该键的通道，而在注销时将从所有选择器的键集中移除该键。 

通过选择操作将键添加到已选择键集中。可通过调用已选择键集的 remove 方法，或者通过调用从该键集获得的 iterator 的 remove 方法直接移除某个键。通过任何其他方式从不会将键从已选择键集中移除；特别是，它们不会因为影响选择操作而被移除。不能将键直接添加到已选择键集中。 

**选择**  
在每次选择操作期间，都可以将键添加到选择器的已选择键集以及从中将其移除，并且可以从其键集和已取消键集中将其移除。选择是由 select()、select(long) 和 selectNow() 方法执行的，执行涉及三个步骤：   

1. 将已取消键集中的每个键从所有键集中移除（如果该键是键集的成员），并注销其通道。此步骤使已取消键集成为空集。 

2. 在开始进行选择操作时，应查询基础操作系统来更新每个剩余通道的准备就绪信息，以执行由其键的相关集合所标识的任意操作。对于已为至少一个这样的操作准备就绪的通道，执行以下两种操作之一： 

	a. 如果该通道的键尚未在已选择键集中，则将其添加到该集合中，并修改其准备就绪操作集，以准确地标识那些通道现在已报告为之准备就绪的操作。丢弃准备就绪操作集中以前记录的所有准备就绪信息。 

	b. 如果该通道的键已经在已选择键集中，则修改其准备就绪操作集，以准确地标识所有通道已报告为之准备就绪的新操作。保留准备就绪操作集以前记录的所有准备就绪信息；换句话说，基础系统所返回的准备就绪操作集是和该键的当前准备就绪操作集按位分开 (bitwise-disjoined) 的。 

3. 如果在此步骤开始时键集中的所有键都有空的相关集合，则不会更新已选择键集和任意键的准备就绪操作集。 
如果在步骤 (2) 的执行过程中要将任意键添加到已取消键集中，则处理过程如步骤 (1)。 

是否阻塞选择操作以等待一个或多个通道准备就绪，如果这样做的话，要等待多久，这是三种选择方法之间的唯一本质差别。 

**并发性**  
选择器自身可由多个并发线程安全使用，但是其键集并非如此。 

选择操作在选择器本身上、在键集上和在已选择键集上是同步的，顺序也与此顺序相同。在执行上面的步骤 (1) 和 (3) 时，它们在已取消键集上也是同步的。 

在执行选择操作的过程中，更改选择器键的相关集合对该操作没有影响；进行下一次选择操作才会看到此更改。 

可在任意时间取消键和关闭通道。因此，在一个或多个选择器的键集中出现某个键并不意味着该键是有效的，也不意味着其通道处于打开状态。如果存在另一个线程取消某个键或关闭某个通道的可能性，那么应用程序代码进行同步时应该小心，并且必要时应该检查这些条件。 

阻塞在 select() 或 select(long) 方法之一中的某个线程可能被其他线程以下列三种方式之一中断： 

● 通过调用选择器的 wakeup 方法，   
● 通过调用选择器的 close 方法，或者  
● 在通过调用已阻塞线程的 interrupt 方法的情况下，将设置其中断状态并且将调用该选择器的 wakeup 方法。 

close 方法在选择器上是同步的，并且所有三个键集都与选择操作中的顺序相同。 

一般情况下，选择器的键和已选择键集由多个并发线程使用是不安全的。如果这样的线程可以直接修改这些键集之一，那么应该通过对该键集本身进行同步来控制访问。

**管理选择键**

通常的做法是在选择器上调用一次select操作(这将更新已选择的键的集合)，然后遍历selectKeys( )方法返回的键的集合。在按顺序进行检查每个键的过程中，相关的通道也根据键的就绪集合进行处理。然后键将从已选择的键的集合中被移除（通过在Iterator对象上调用remove( )方法），然后检查下一个键。完成后，通过再次调用select( )方法重复这个循环。

示例：

	public class SelectSockets {
	
		public static int port_number = 1234;
	
		public static void main(String[] args) throws IOException {
			new SelectSockets().start(args);
		}
	
		public void start(String[] args) throws IOException{
			int port = port_number;
			if(args.length > 0){
				port = Integer.parseInt(args[0]);
			}
	
			ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
			ServerSocket serverSocket = serverSocketChannel.socket();
			serverSocket.bind(new InetSocketAddress(port));
			serverSocketChannel.configureBlocking(false);
	
			Selector selector = Selector.open();
			serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
	
			while(true){
				int n = selector.select();
				if(n == 0){
					continue;
				}
	
				Iterator<SelectionKey>  iter = selector.selectedKeys().iterator();
				while(iter.hasNext()){
					SelectionKey key = iter.next();
					if(key.isAcceptable()){
	
						ServerSocketChannel server = (ServerSocketChannel)key.channel();
						SocketChannel channel = server.accept();
						registerChannel(selector,channel,SelectionKey.OP_READ);
						sayHello(channel);
					}
	
					if(key.isReadable()){
						readDataFromSocket(key);
					}
					iter.remove();
				}
			}
		}
	
		public ByteBuffer buffer = ByteBuffer.allocate(1024);
	
		private void readDataFromSocket(SelectionKey key) throws IOException {
			SocketChannel socketChannel = (SocketChannel) key.channel();
			int count;
	
			buffer.clear();
			while( (count =socketChannel.read(buffer)) > 0){
				buffer.flip();
	
				while (buffer.hasRemaining()) {
					socketChannel.write(buffer);
				}
				buffer.clear();
			}
	
			if(count < 0){
				socketChannel.close();
			}
		}
	
		private void sayHello(SocketChannel channel) throws IOException {
			buffer.clear();
			buffer.put("Hi here! \r\n".getBytes());
			buffer.flip();
	
			channel.write(buffer);
		}
	
		private void registerChannel(Selector selector, SocketChannel channel,int ops) throws IOException {
			if(channel == null){
				return;
			}
			channel.configureBlocking(false);
			channel.register(selector, ops);
		}
	
	}

参考资料

- JDK
- Java NIO
- [Java NIO Selector](http://tutorials.jenkov.com/java-nio/selectors.html) - 一篇很好的介绍selector的文章

