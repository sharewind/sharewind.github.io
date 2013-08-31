---
layout: post
title: "thrift async client"
date: 2013-07-30 21:26
comments: true
categories: thrift
---

####一、Java的thrift异步Client

#####1.异步的thrift client类 TAsyncClient

{% codeblock TAsyncClient.java lang:java %}
 
	public abstract class TAsyncClient {
	  protected final TProtocolFactory ___protocolFactory;
	  protected final TNonblockingTransport ___transport;
	  protected final TAsyncClientManager ___manager;
	  protected TAsyncMethodCall ___currentMethod;
	  private Exception ___error;
	  private long ___timeout;
	
	  public TAsyncClient(TProtocolFactory protocolFactory, TAsyncClientManager manager, TNonblockingTransport transport) {
	    this(protocolFactory, manager, transport, 0);
	  }
	
	  public TAsyncClient(TProtocolFactory protocolFactory, TAsyncClientManager manager, TNonblockingTransport transport, long timeout) {
	    this.___protocolFactory = protocolFactory;
	    this.___manager = manager;
	    this.___transport = transport;
	    this.___timeout = timeout;
	  }
	
	  public TProtocolFactory getProtocolFactory() {
	    return ___protocolFactory;
	  }
	
	  public long getTimeout() {
	    return ___timeout;
	  }
	
	  public boolean hasTimeout() {
	    return ___timeout > 0;
	  }
	
	  public void setTimeout(long timeout) {
	    this.___timeout = timeout;
	  }
	
	  /**
	   * Is the client in an error state?
	   * @return If client in an error state?
	   */
	  public boolean hasError() {
	    return ___error != null;
	  }
	
	  /**
	   * Get the client's error - returns null if no error
	   * @return Get the client's error. <br /> returns null if no error
	   */
	  public Exception getError() {
	    return ___error;
	  }
	
	  protected void checkReady() {
		// 同时只能调用一个方法
	    // Ensure we are not currently executing a method
	    if (___currentMethod != null) {
	      throw new IllegalStateException("Client is currently executing another method: " + ___currentMethod.getClass().getName());
	    }
	
	    // Ensure we're not in an error state
	    if (___error != null) {
	      throw new IllegalStateException("Client has an error!", ___error);
	    }
	  }
	
	  /**
	   * Called by delegate method when finished
	   */
	  protected void onComplete() {
		// 当前方法完成
	    ___currentMethod = null;
	  }
	
	  /**
	   * Called by delegate method on error
	   */
	  protected void onError(Exception exception) {
		// 发生错误，关闭连接，赋值给__error
	    ___transport.close();
	    ___currentMethod = null;
	    ___error = exception;
	  }
	}

{% endcodeblock %}


##### 2.异步方法调用 

{% codeblock TAsyncMethodCall.java lang:java %}
	
	public abstract class TAsyncMethodCall<T> {
	
	  private static final int INITIAL_MEMORY_BUFFER_SIZE = 128;
	  private static AtomicLong sequenceIdCounter = new AtomicLong(0);
	
	  public static enum State {
	    CONNECTING,
	    WRITING_REQUEST_SIZE,
	    WRITING_REQUEST_BODY,
	    READING_RESPONSE_SIZE,
	    READING_RESPONSE_BODY,
	    RESPONSE_READ,
	    ERROR;
	  }
	
	  /**
	   * Next step in the call, initialized by start()
	   */
	  private State state = null;
	
	  protected final TNonblockingTransport transport;
	  private final TProtocolFactory protocolFactory;
	  protected final TAsyncClient client;
	  private final AsyncMethodCallback<T> callback;
	  private final boolean isOneway;
	  private long sequenceId;
	  
	  private ByteBuffer sizeBuffer;
	  private final byte[] sizeBufferArray = new byte[4];
	  private ByteBuffer frameBuffer;
	
	  private long startTime = System.currentTimeMillis();
	
	  protected TAsyncMethodCall(TAsyncClient client, TProtocolFactory protocolFactory, TNonblockingTransport transport, AsyncMethodCallback<T> callback, boolean isOneway) {
	    this.transport = transport;
	    this.callback = callback;
	    this.protocolFactory = protocolFactory;
	    this.client = client;
	    this.isOneway = isOneway;
	    this.sequenceId = TAsyncMethodCall.sequenceIdCounter.getAndIncrement();
	  }
	
	  protected State getState() {
	    return state;
	  }
	
	  protected boolean isFinished() {
	    return state == State.RESPONSE_READ;
	  }
	
	  protected long getStartTime() {
	    return startTime;
	  }
	  
	  protected long getSequenceId() {
	    return sequenceId;
	  }
	
	  public TAsyncClient getClient() {
	    return client;
	  }
	  
	  public boolean hasTimeout() {
	    return client.hasTimeout();
	  }
	  
	  public long getTimeoutTimestamp() {
	    return client.getTimeout() + startTime;
	  }
	
	  // 每个具体的方法调用会在这里面写入请求的方法名等信息。。。。
	  protected abstract void write_args(TProtocol protocol) throws TException;
	
	  // 准备方法调用：初始化缓存，写入参数到缓存中
	  /**
	   * Initialize buffers.
	   * @throws TException if buffer initialization fails
	   */
	  protected void prepareMethodCall() throws TException {
		// 准备方法调用写入参数
	    TMemoryBuffer memoryBuffer = new TMemoryBuffer(INITIAL_MEMORY_BUFFER_SIZE);
	    TProtocol protocol = protocolFactory.getProtocol(memoryBuffer);
	    
		// 写入方法的请求参数
		write_args(protocol);
	
	    int length = memoryBuffer.length();
	    frameBuffer = ByteBuffer.wrap(memoryBuffer.getArray(), 0, length);
	
		// 写入frame size 和 实现内容
	    TFramedTransport.encodeFrameSize(length, sizeBufferArray);
	    sizeBuffer = ByteBuffer.wrap(sizeBufferArray);
	  }
	
	  /**
	   * Register with selector and start first state, which could be either connecting or writing.
	   * @throws IOException if register or starting fails
	   */
	  void start(Selector sel) throws IOException {
	    SelectionKey key;
	    if (transport.isOpen()) {
		  // 写入请求大小
	      state = State.WRITING_REQUEST_SIZE;
		  // 注册到选择器上
	      key = transport.registerSelector(sel, SelectionKey.OP_WRITE);
	    } else {
		  // 建立连接
	      state = State.CONNECTING;
		  // 注册到选择器上
	      key = transport.registerSelector(sel, SelectionKey.OP_CONNECT);
	
	      // non-blocking connect can complete immediately,
	      // in which case we should not expect the OP_CONNECT
	      if (transport.startConnect()) {
	        registerForFirstWrite(key);
	      }
	    }
	
		// 附上TAsyncMethodCall自身
	    key.attach(this);
	  }
	
	  void registerForFirstWrite(SelectionKey key) throws IOException {
	    state = State.WRITING_REQUEST_SIZE;
	    key.interestOps(SelectionKey.OP_WRITE);
	  }
	
	  protected ByteBuffer getFrameBuffer() {
	    return frameBuffer;
	  }


	 // 迁移到下一个状态	
	 /**
	   * Transition to next state, doing whatever work is required. Since this
	   * method is only called by the selector thread, we can make changes to our
	   * select interests without worrying about concurrency.
	   * @param key
	   */
	  protected void transition(SelectionKey key) {
	    // Ensure key is valid
	    if (!key.isValid()) {
	      key.cancel();
	      Exception e = new TTransportException("Selection key not valid!");
	      onError(e);
	      return;
	    }
	
	    // Transition function
	    try {
	      switch (state) {
	        case CONNECTING:
	          doConnecting(key);
	          break;
	        case WRITING_REQUEST_SIZE:
	          doWritingRequestSize();
	          break;
	        case WRITING_REQUEST_BODY:
	          doWritingRequestBody(key);
	          break;
	        case READING_RESPONSE_SIZE:
	          doReadingResponseSize();
	          break;
	        case READING_RESPONSE_BODY:
	          doReadingResponseBody(key);
	          break;
	        default: // RESPONSE_READ, ERROR, or bug
	          throw new IllegalStateException("Method call in state " + state
	              + " but selector called transition method. Seems like a bug...");
	      }
	    } catch (Exception e) {
	      key.cancel();
	      key.attach(null);
	      onError(e);
	    }
	  }
	
	  protected void onError(Exception e) {
	    client.onError(e);
	    callback.onError(e);
	    state = State.ERROR;
	  }
	
	  private void doReadingResponseBody(SelectionKey key) throws IOException {
	    if (transport.read(frameBuffer) < 0) {
	      throw new IOException("Read call frame failed");
	    }
	    if (frameBuffer.remaining() == 0) {
	      cleanUpAndFireCallback(key);
	    }
	  }
	
	  private void cleanUpAndFireCallback(SelectionKey key) {
	    state = State.RESPONSE_READ;
	    key.interestOps(0);
	    // this ensures that the TAsyncMethod instance doesn't hang around
	    key.attach(null);
	    client.onComplete();
	    callback.onComplete((T)this);
	  }
	
	  private void doReadingResponseSize() throws IOException {
	    if (transport.read(sizeBuffer) < 0) {
	      throw new IOException("Read call frame size failed");
	    }
	    if (sizeBuffer.remaining() == 0) {
	      state = State.READING_RESPONSE_BODY;
	      frameBuffer = ByteBuffer.allocate(TFramedTransport.decodeFrameSize(sizeBufferArray));
	    }
	  }
	
	  private void doWritingRequestBody(SelectionKey key) throws IOException {
	    if (transport.write(frameBuffer) < 0) {
	      throw new IOException("Write call frame failed");
	    }
	    if (frameBuffer.remaining() == 0) {
	      if (isOneway) {
	        cleanUpAndFireCallback(key);
	      } else {
	        state = State.READING_RESPONSE_SIZE;
	        sizeBuffer.rewind();  // Prepare to read incoming frame size
	        key.interestOps(SelectionKey.OP_READ);
	      }
	    }
	  }
	
	  private void doWritingRequestSize() throws IOException {
	    if (transport.write(sizeBuffer) < 0) {
	      throw new IOException("Write call frame size failed");
	    }
	    if (sizeBuffer.remaining() == 0) {
	      state = State.WRITING_REQUEST_BODY;
	    }
	  }
	
	  private void doConnecting(SelectionKey key) throws IOException {
	    if (!key.isConnectable() || !transport.finishConnect()) {
	      throw new IOException("not connectable or finishConnect returned false after we got an OP_CONNECT");
	    }
	    registerForFirstWrite(key);
	  }
	}

{% endcodeblock %}


##### 3. TAsyncClientManager

可以同时支持多个client的异步调用，相当于客户端的NIO selector 和线程管理。

{% codeblock TAsyncClientManager.java lang:java %}

	/**
	 * Contains selector thread which transitions method call objects
	 */
	public class TAsyncClientManager {
	  private static final Logger LOGGER = LoggerFactory.getLogger(TAsyncClientManager.class.getName());
	
	  private final SelectThread selectThread;
	  private final ConcurrentLinkedQueue<TAsyncMethodCall> pendingCalls = new ConcurrentLinkedQueue<TAsyncMethodCall>();
	
	  public TAsyncClientManager() throws IOException {
		// 构造时即新建一个SelectThread线程
	    this.selectThread = new SelectThread();
	    selectThread.start();
	  }
	
	  // 进行方法调用的入口
	  public void call(TAsyncMethodCall method) throws TException {
	    if (!isRunning()) {
	      throw new TException("SelectThread is not running");
	    }
		// 准备方法调用，写入参数等
	    method.prepareMethodCall();
		// 加入等待队列
	    pendingCalls.add(method);
		// 立即唤醒selector, select 阻塞中会立即返回
	    selectThread.getSelector().wakeup();
	  }
	
	  public void stop() {
	    selectThread.finish();
	  }
	
	  public boolean isRunning() {
	    return selectThread.isAlive();
	  }
	
	  private class SelectThread extends Thread {
	    private final Selector selector;
	    private volatile boolean running;
	    private final TreeSet<TAsyncMethodCall> timeoutWatchSet = new TreeSet<TAsyncMethodCall>(new TAsyncMethodCallTimeoutComparator());
	
	    public SelectThread() throws IOException {
		  // 创建selector 
	      this.selector = SelectorProvider.provider().openSelector();
	      this.running = true;
	      this.setName("TAsyncClientManager#SelectorThread " + this.getId());
	
	      // We don't want to hold up the JVM when shutting down
	      setDaemon(true);
	    }
	
	    public Selector getSelector() {
	      return selector;
	    }
	
	    public void finish() {
	      running = false;
	      selector.wakeup();
	    }
	
	    public void run() {
	      while (running) {
	        try {
			  // 进行select操作
	          try {
	            if (timeoutWatchSet.size() == 0) {
	              // No timeouts, so select indefinitely
	              selector.select();
	            } else {
	              // We have a timeout pending, so calculate the time until then and select appropriately
	              long nextTimeout = timeoutWatchSet.first().getTimeoutTimestamp();
	              long selectTime = nextTimeout - System.currentTimeMillis();
	              if (selectTime > 0) {
	                // Next timeout is in the future, select and wake up then
	                selector.select(selectTime);
	              } else {
	                // Next timeout is now or in past, select immediately so we can time out
	                selector.selectNow();
	              }
	            }
	          } catch (IOException e) {
	            LOGGER.error("Caught IOException in TAsyncClientManager!", e);
	          }
			  // 迁移方法状态 
	          transitionMethods();
			  // 判断方法过期 
	          timeoutMethods();
			  // 开始那些等待的方法
	          startPendingMethods();
	        } catch (Exception exception) {
	          LOGGER.error("Ignoring uncaught exception in SelectThread", exception);
	        }
	      }
	    }
	
	    // Transition methods for ready keys
	    private void transitionMethods() {
	      try {
			// 处理各种就绪的keys 
	        Iterator<SelectionKey> keys = selector.selectedKeys().iterator();
	        while (keys.hasNext()) {
	          SelectionKey key = keys.next();
	          keys.remove();
	          if (!key.isValid()) {
	            // this can happen if the method call experienced an error and the
	            // key was cancelled. can also happen if we timeout a method, which
	            // results in a channel close.
	            // just skip
	            continue;
	          }
	          TAsyncMethodCall methodCall = (TAsyncMethodCall)key.attachment();
	          methodCall.transition(key);
	
			  // 执行完成或发生错误，从timeout 中移出
	          // If done or error occurred, remove from timeout watch set
	          if (methodCall.isFinished() || methodCall.getClient().hasError()) {
	            timeoutWatchSet.remove(methodCall);
	          }
	        }
	      } catch (ClosedSelectorException e) {
	        LOGGER.error("Caught ClosedSelectorException in TAsyncClientManager!", e);
	      }
	    }
	
		// 判断方法是否timeout
	    // Timeout any existing method calls
	    private void timeoutMethods() {
	      Iterator<TAsyncMethodCall> iterator = timeoutWatchSet.iterator();
	      long currentTime = System.currentTimeMillis();
	      while (iterator.hasNext()) {
	        TAsyncMethodCall methodCall = iterator.next();
	        if (currentTime >= methodCall.getTimeoutTimestamp()) {
	          iterator.remove();
	          methodCall.onError(new TimeoutException("Operation " + methodCall.getClass() + " timed out after " + (currentTime - methodCall.getStartTime()) + " ms."));
	        } else {
	          break;
	        }
	      }
	    }
	
	    // Start any new calls
	    private void startPendingMethods() {
	      TAsyncMethodCall methodCall;
	      while ((methodCall = pendingCalls.poll()) != null) {
	        // Catch registration errors. method will catch transition errors and cleanup.
	        try {
			  // 开始执行方法，注册到selector上。
	          methodCall.start(selector);
	
			  // 加入timeout 监测
	          // If timeout specified and first transition went smoothly, add to timeout watch set
	          TAsyncClient client = methodCall.getClient();
	          if (client.hasTimeout() && !client.hasError()) {
	            timeoutWatchSet.add(methodCall);
	          }
	        } catch (Exception exception) {
	          LOGGER.warn("Caught exception in TAsyncClientManager!", exception);
	          methodCall.onError(exception);
	        }
	      }
	    }
	  }
	
	  /** Comparator used in TreeSet */
	  private static class TAsyncMethodCallTimeoutComparator implements Comparator<TAsyncMethodCall> {
	    public int compare(TAsyncMethodCall left, TAsyncMethodCall right) {
	      if (left.getTimeoutTimestamp() == right.getTimeoutTimestamp()) {
	        return (int)(left.getSequenceId() - right.getSequenceId());
	      } else {
	        return (int)(left.getTimeoutTimestamp() - right.getTimeoutTimestamp());
	      }
	    }
	  }
	}
{% endcodeblock %}


##### 4. TAsyncMethodCall 具体实现示例


{% codeblock TAsyncMethodCall-add_call lang:java %}

    public static class add_call extends org.apache.thrift.async.TAsyncMethodCall {

      private long a;
      private long b;
      public add_call(long a, long b, org.apache.thrift.async.AsyncMethodCallback<add_call> resultHandler, org.apache.thrift.async.TAsyncClient client, org.apache.thrift.protocol.TProtocolFactory protocolFactory, org.apache.thrift.transport.TNonblockingTransport transport) throws org.apache.thrift.TException {
        super(client, protocolFactory, transport, resultHandler, false);
        this.a = a;
        this.b = b;
      }

	  // 这个方法实现 写入请求的参数
      public void write_args(org.apache.thrift.protocol.TProtocol prot) throws org.apache.thrift.TException {
		// 写入请求的方法名
        prot.writeMessageBegin(new org.apache.thrift.protocol.TMessage("add", org.apache.thrift.protocol.TMessageType.CALL, 0));
        add_args args = new add_args();
        args.setA(a);
        args.setB(b);
        args.write(prot);
        prot.writeMessageEnd();
      }

{% endcodeblock %}


#### 二、Python的异步调用Client

##### 1. 生成的Python接口文件

{% codeblock OAuthProviderService.py-Iface lang:python %}

	class Iface(object):
	  def registerApp(self, app, callback):
	    """
	    Parameters:
	     - app
	    """
	    pass

{% endcodeblock %}

##### 2. 生成的Client实现类

{% codeblock OAuthProviderService.py-Client lang:python %}

	class Client(Iface):
	  def __init__(self, transport, iprot_factory, oprot_factory=None):
	    self._transport = transport
	    self._iprot_factory = iprot_factory
	    self._oprot_factory = (oprot_factory if oprot_factory is not None
	                           else iprot_factory)
		# 初始化请求的id
	    self._seqid = 0
	    self._reqs = {}
	
      # 请求回调时的处理类
	  @gen.engine
	  def recv_dispatch(self):
	    """read a response from the wire. schedule exactly one per send that
	    expects a response, but it doesn't matter which callee gets which
	    response; they're dispatched here properly"""
	
		#等待数据返回
		#发送请求A
		#发送请求B
		#有可能发生：先返回请求B，再返回请求A的情景，所以根据seqid等信息来获取相应的callback方法
		
		# 等待网络IO有数据返回，并且读取到frame
	    # wait for a frame header
	    frame = yield gen.Task(self._transport.readFrame)
		#放入内存TTransport	
	    tr = TTransport.TMemoryBuffer(frame)
		#读取方法名称，消息类型，返回的reqid
	    iprot = self._iprot_factory.getProtocol(tr)
	    (fname, mtype, rseqid) = iprot.readMessageBegin()
		#通过反射获取方法
	    method = getattr(self, 'recv_' + fname)
		#调用回调方法
	    method(iprot, mtype, rseqid)
	
	  #实际的业务方法：注册App
	  def registerApp(self, app, callback):
	    """
	    Parameters:
	     - app
	    """
	    self._seqid += 1 #seqid加1
		#将回调方法，放入类的请求队列中，相当于一个map
	    self._reqs[self._seqid] = callback
		#发送数据请求
	    self.send_registerApp(app)
		#等待请求返回
	    self.recv_dispatch()
	
	  def send_registerApp(self, app):
		#获取输出协议
	    oprot = self._oprot_factory.getProtocol(self._transport)
		#写入消息头：调用方法、消息类型、seqId
		#写入请求参数：方法的调用参数
		#flush transport的数据内容到网络 
	    oprot.writeMessageBegin('registerApp', TMessageType.CALL, self._seqid)
	    args = registerApp_args()
	    args.app = app
	    args.write(oprot)
	    oprot.writeMessageEnd()
	    oprot.trans.flush()
	
	  def recv_registerApp(self, iprot, mtype, rseqid):
		#根据rseqid从请求队列中获取callback回调方法
	    callback = self._reqs.pop(rseqid)
		#根据返回的消息类型进行相应处理
	    if mtype == TMessageType.EXCEPTION:
	      x = TApplicationException()
	      x.read(iprot)
	      iprot.readMessageEnd()
	      callback(x)
	      return
	    result = registerApp_result()
	    result.read(iprot)
	    iprot.readMessageEnd()
	    if result.success is not None:
	      callback(result.success)
	      return
	    callback(TApplicationException(TApplicationException.MISSING_RESULT, "registerApp failed: unknown result"))
	    return

{% endcodeblock %}


##### 3. 与tornado的ioloop 结合在一起的 TTornadoStreamTransport

{% codeblock TTornadoStreamTransport.py lang:python %}

	class TTornadoStreamTransport(TTransport.TTransportBase):
	    
		"""a framed, buffered transport over a Tornado stream"""
		#构造函数，传入目标的host,port
	    def __init__(self, host, port, stream=None):
	        self.host = host
	        self.port = port
	        self.is_queuing_reads = False
	        self.read_queue = []
	        self.__wbuf = StringIO()
	
	        # servers provide a ready-to-go stream
	        self.stream = stream
	        if self.stream is not None:
	            self._set_close_callback()
	
		#打开transport,（进行网络连接）
	    # not the same number of parameters as TTransportBase.open
	    def open(self, callback):
			#创建一个sock
	        logging.debug('socket connecting')
	        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM, 0)
	        
			#构造tornado的iostream对象
			self.stream = iostream.IOStream(sock)
	
			#连接失败的回调函数
	        def on_close_in_connect(*_):
	            message = 'could not connect to %s:%s' % (self.host, self.port)
	            raise TTransportException(
	                type=TTransportException.NOT_OPEN,
	                message=message)
	        self.stream.set_close_callback(on_close_in_connect)
	
			#连接成功的回调函数
	        def finish(*_):
	            self._set_close_callback()
	            callback()
	
	        self.stream.connect((self.host, self.port), callback=finish)
	
	    def _set_close_callback(self):
			#这里定义的这个方法有啥用？？？？
	        def on_close():
	            raise TTransportException(
	                type=TTransportException.END_OF_FILE,
	                message='socket closed')
	        self.stream.set_close_callback(self.close)
	
	    def close(self):
	        # don't raise if we intend to close
	        self.stream.set_close_callback(None)
	        self.stream.close()
	
		#从不进行单独的读取操作，每次只能读取一个frame
	    def read(self, _):
	        # The generated code for Tornado shouldn't do individual reads -- only
	        # frames at a time
	        assert "you're doing it wrong" is True
	
	    @gen.engine
	    def readFrame(self, callback):
	        self.read_queue.append(callback)
	        logging.debug('read queue: %s', self.read_queue)
	
	        if self.is_queuing_reads:
	            # If a read is already in flight, then the while loop below should
	            # pull it from self.read_queue
	            return
	
	        self.is_queuing_reads = True
	        while self.read_queue:
	            next_callback = self.read_queue.pop()
	            result = yield gen.Task(self._readFrameFromStream)
	            next_callback(result)
	        self.is_queuing_reads = False
	
		#从数据流中读取数据帧
	    @gen.engine
	    def _readFrameFromStream(self, callback):
	        logging.debug('_readFrameFromStream')
			#读取帧头得到帧的长度
	        frame_header = yield gen.Task(self.stream.read_bytes, 4)
	        frame_length, = struct.unpack('!i', frame_header)
	        logging.debug('received frame header, frame length = %i', frame_length)
			#读完整个帧
	        frame = yield gen.Task(self.stream.read_bytes, frame_length)
	        logging.debug('received frame payload')
	        callback(frame)
	
		#写入到当前的buffer中
	    def write(self, buf):
	        self.__wbuf.write(buf)
	
	    def flush(self, callback=None):
			#得到输出的缓冲内容
	        wout = self.__wbuf.getvalue()
			#输出内容的长度	        
			wsz = len(wout)

			#frame的数据格式：4个字节的整数表示帧大小|帧的数据内容|

	        # reset wbuf before write/flush to preserve state on underlying failure
	        self.__wbuf = StringIO()
	        # N.B.: Doing this string concatenation is WAY cheaper than making
	        # two separate calls to the underlying socket object. Socket writes in
	        # Python turn out to be REALLY expensive, but it seems to do a pretty
	        # good job of managing string buffer operations without excessive copies
	        buf = struct.pack("!i", wsz) + wout
	
	        logging.debug('writing frame length = %i', wsz)
	        self.stream.write(buf, callback)

{% endcodeblock %}


##### 4. tornado与thrift 结合client 的包装器

{% codeblock ClientWrapper lang:python %}

	class Client():
	    """
	        Thrift Client proxying thrift methods defined on `iface_cls`.
	        A simple load balancer added in.
	        the MISSING_RESULT exception will be changed None to calller.
	    
	    """    
	    def __init__(self, iface_cls, servers):
	        self._iface_cls = iface_cls
	        self._servers = servers
	
	
	    def __getattr__(self, attr):
	        @gen.engine
	        def client_call(*args, **kwargs):
	            server = self._find_server()
	            host, port = server.split(":")
	            transport = cloudatlas.thrift.TTornado.TTornadoStreamTransport(host, int(port))
	            pfactory = TBinaryProtocol.TBinaryProtocolFactory()
	            _client = self._iface_cls(transport, pfactory) 
	                     
	            try:
	                yield gen.Task(transport.open)
	        
	                _callback = kwargs['callback']
	                del(kwargs['callback'])
	
	                result = yield gen.Task(getattr(_client, attr), *args, **kwargs)
	                #print result
	                if type(result) == Thrift.TApplicationException and result.type == Thrift.TApplicationException.MISSING_RESULT:
	                    result = None # ---------------------- hacking for return None object 
	                _client._transport.close()
	                _callback(result)
	            except TTransport.TTransportException as e:
	                _client._transport.close()
	                raise
	            except Exception as e:
	                _client._transport.close()
	                raise
	
	        setattr(self, attr, client_call)
	        return getattr(self, attr)
	
	
	
	    def _find_server(self):
	        ''' no round robin, just random choose a server '''
	        return choice(self._servers)

{% endcodeblock %}