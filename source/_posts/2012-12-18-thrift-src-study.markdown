---
layout: post
title: "thrift src study"
date: 2012-12-18 21:26
comments: true
categories: thrift
---

{% codeblock SelectAcceptThread-接收请求的线程 lang:java %}

	protected class SelectAcceptThread extends AbstractSelectThread {

    // The server transport on which new client transports will be accepted
    private final TNonblockingServerTransport serverTransport;

    /**
     * Set up the thread that will handle the non-blocking accepts, reads, and
     * writes.
     */
    public SelectAcceptThread(final TNonblockingServerTransport serverTransport)
    throws IOException {
      this.serverTransport = serverTransport;
      serverTransport.registerSelector(selector);
    }

    public boolean isStopped() {
      return stopped_;
    }

    /**
     * The work loop. Handles both selecting (all IO operations) and managing
     * the selection preferences of all existing connections.
     */
    public void run() {
      try {
        while (!stopped_) {
		  // 这里执行select 方法
          select();
		  // 处理NIO事件
          processInterestChanges();
        }
        for (SelectionKey selectionKey : selector.keys()) {
          cleanupSelectionKey(selectionKey);
        }
      } catch (Throwable t) {
        LOGGER.error("run() exiting due to uncaught error", t);
      } finally {
        stopped_ = true;
      }
    }
{% endcodeblock %}

{% codeblock SelectAcceptThread-select方法 lang:java %}

    private void select() {
      try {
        // wait for io events.
        selector.select();

        // process the io events we received
        Iterator<SelectionKey> selectedKeys = selector.selectedKeys().iterator();
        while (!stopped_ && selectedKeys.hasNext()) {
          SelectionKey key = selectedKeys.next();
          selectedKeys.remove();

          // skip if not valid
          if (!key.isValid()) {
            cleanupSelectionKey(key);
            continue;
          }

		  // 分别处理感兴趣的事件
          // if the key is marked Accept, then it has to be the server
          // transport.
          if (key.isAcceptable()) {
            handleAccept();
          } else if (key.isReadable()) {
            // deal with reads
            handleRead(key);
          } else if (key.isWritable()) {
            // deal with writes
            handleWrite(key);
          } else {
            LOGGER.warn("Unexpected state in select! " + key.interestOps());
          }
        }
      } catch (IOException e) {
        LOGGER.warn("Got an IOException while selecting!", e);
      }
    }


	// 这个方法会在后面的调用中被用到。。。
    /**
     * Check to see if there are any FrameBuffers that have switched their
     * interest type from read to write or vice versa.
     */
    protected void processInterestChanges() {
      synchronized (selectInterestChanges) {
        for (FrameBuffer fb : selectInterestChanges) {
          fb.changeSelectInterests();
        }
        selectInterestChanges.clear();
      }
    }

{% endcodeblock %}


{% codeblock SelectAcceptThread-接收一个新的连接 lang:java %}


    /**
     * Accept a new connection.
     */
    private void handleAccept() throws IOException {
      SelectionKey clientKey = null;
      TNonblockingTransport client = null;
      try {
        // accept the connection
        client = (TNonblockingTransport)serverTransport.accept();
        clientKey = client.registerSelector(selector, SelectionKey.OP_READ);

		// 每次都新建一个FrameBuffer，
        // add this key to the map
        FrameBuffer frameBuffer = new FrameBuffer(client, clientKey,
          SelectAcceptThread.this);
        clientKey.attach(frameBuffer);
      } catch (TTransportException tte) {
        // something went wrong accepting.
        LOGGER.warn("Exception trying to accept!", tte);
        tte.printStackTrace();
        if (clientKey != null) cleanupSelectionKey(clientKey);
        if (client != null) client.close();
      }
    }
{% endcodeblock %}


{% codeblock FrameBuffer的构造函数. lang:java %}
		
    public FrameBuffer(final TNonblockingTransport trans,
        final SelectionKey selectionKey,
        final AbstractSelectThread selectThread) {
      trans_ = trans;
      selectionKey_ = selectionKey;
      selectThread_ = selectThread;
	  // 初始化的Buffer 为4个字节，用来读取FrameSize 的整数
      buffer_ = ByteBuffer.allocate(4);
    }

	// Frame 的初始状态 
    // where in the process of reading/writing are we?
    private FrameBufferState state_ = FrameBufferState.READING_FRAME_SIZE;
{% endcodeblock %}



{% codeblock SelectAcceptThread-handleRead. lang:java %}

    /**
     * Do the work required to read from a readable client. If the frame is
     * fully read, then invoke the method call.
     */
    protected void handleRead(SelectionKey key) {
      FrameBuffer buffer = (FrameBuffer) key.attachment();
      if (!buffer.read()) {
		// 如果读取失败，清理掉这个selection key 
        cleanupSelectionKey(key);
        return;
      }

	  // 如果整个frame已读取完成，开始调用具体的业务方法
      // if the buffer's frame read is complete, invoke the method.
      if (buffer.isFrameFullyRead()) {
        if (!requestInvoke(buffer)) {
          cleanupSelectionKey(key);
        }
      }
    }

    /**
     * Do connection-close cleanup on a given SelectionKey.
     */
    protected void cleanupSelectionKey(SelectionKey key) {
      // remove the records from the two maps
      FrameBuffer buffer = (FrameBuffer) key.attachment();
      if (buffer != null) {
        // close the buffer
        buffer.close();
      }
      // cancel the selection key
      key.cancel();
    }
{% endcodeblock %}

#####read in FrameBuffer

{% codeblock FrameBuffer-read方法. lang:java %}

    public boolean read() {
      if (state_ == FrameBufferState.READING_FRAME_SIZE) {
        // try to read the frame size completely
        if (!internalRead()) {
          return false;
        }

        // if the frame size has been read completely, then prepare to read the
        // actual frame.
        if (buffer_.remaining() == 0) {
		  // 读取到帧大小
          // pull out the frame size as an integer.
          int frameSize = buffer_.getInt(0);
          if (frameSize <= 0) {
            LOGGER.error("Read an invalid frame size of " + frameSize
                + ". Are you using TFramedTransport on the client side?");
            return false;
          }

          // if this frame will always be too large for this server, log the
          // error and close the connection.
          if (frameSize > MAX_READ_BUFFER_BYTES) {
            LOGGER.error("Read a frame size of " + frameSize
                + ", which is bigger than the maximum allowable buffer size for ALL connections.");
            return false;
          }

          // if this frame will push us over the memory limit, then return.
          // with luck, more memory will free up the next time around.
          if (readBufferBytesAllocated.get() + frameSize > MAX_READ_BUFFER_BYTES) {
            return true;
          }

          // increment the amount of memory allocated to read buffers
          readBufferBytesAllocated.addAndGet(frameSize + 4);
		
		  // 重新分配缓冲区大小
          // reallocate the readbuffer as a frame-sized buffer
          buffer_ = ByteBuffer.allocate(frameSize + 4);
          buffer_.putInt(frameSize);

          state_ = FrameBufferState.READING_FRAME;
        } else {
          // this skips the check of READING_FRAME state below, since we can't
          // possibly go on to that state if there's data left to be read at
          // this one.
          return true;
        }
      }

      // it is possible to fall through from the READING_FRAME_SIZE section
      // to READING_FRAME if there's already some frame data available once
      // READING_FRAME_SIZE is complete.

      if (state_ == FrameBufferState.READING_FRAME) {
        if (!internalRead()) {
          return false;
        }

	    // 读完整个Frame,将当前interestOps 置为0
        // since we're already in the select loop here for sure, we can just
        // modify our selection key directly.
        if (buffer_.remaining() == 0) {
          // get rid of the read select interests
          selectionKey_.interestOps(0);
          state_ = FrameBufferState.READ_FRAME_COMPLETE;
        }

        return true;
      }

      // if we fall through to this point, then the state must be invalid.
      LOGGER.error("Read was called but state is invalid (" + state_ + ")");
      return false;
    }

    /**
     * Check if this FrameBuffer has a full frame read.
     */
    public boolean isFrameFullyRead() {
      return state_ == FrameBufferState.READ_FRAME_COMPLETE;
    }


	// 关闭网络连接
    /**
     * Shut the connection down.
     */
    public void close() {
      // if we're being closed due to an error, we might have allocated a
      // buffer that we need to subtract for our memory accounting.
      if (state_ == FrameBufferState.READING_FRAME || state_ == FrameBufferState.READ_FRAME_COMPLETE) {
        readBufferBytesAllocated.addAndGet(-buffer_.array().length);
      }
      trans_.close();
    }
{% endcodeblock %}


#####invoke in FrameBuffer 

{% codeblock FrameBuffer-invoke 进行实际调用. lang:java %}

    /**
     * Actually invoke the method signified by this FrameBuffer.
     */
    public void invoke() {
      TTransport inTrans = getInputTransport();
      TProtocol inProt = inputProtocolFactory_.getProtocol(inTrans);
      TProtocol outProt = outputProtocolFactory_.getProtocol(getOutputTransport());

      try {
        processorFactory_.getProcessor(inTrans).process(inProt, outProt);
		// 准备输出 
        responseReady();
        return;
      } catch (TException te) {
        LOGGER.warn("Exception while invoking!", te);
      } catch (Throwable t) {
        LOGGER.error("Unexpected throwable while invoking!", t);
      }
      // This will only be reached when there is a throwable.
      state_ = FrameBufferState.AWAITING_CLOSE;
      requestSelectInterestChange();
    }


	// 输出时放在byteArrayOutputStream 中
    private TTransport getOutputTransport() {
      response_ = new TByteArrayOutputStream();
      return outputTransportFactory_.getTransport(new TIOStreamTransport(response_));
    }

    public void responseReady() {
      // the read buffer is definitely no longer in use, so we will decrement
      // our read buffer count. we do this here as well as in close because
      // we'd like to free this read memory up as quickly as possible for other
      // clients.
      readBufferBytesAllocated.addAndGet(-buffer_.array().length);

	  // 没有返回值的方法
      if (response_.len() == 0) {
        // 状态为等待注册为读取。。。
        // go straight to reading again. this was probably an oneway method
        state_ = FrameBufferState.AWAITING_REGISTER_READ;
        buffer_ = null;
      } else {
		// 把response 放入buffer中，并更新状态为等待写入
        buffer_ = ByteBuffer.wrap(response_.get(), 0, response_.len());

        // set state that we're waiting to be switched to write. we do this
        // asynchronously through requestSelectInterestChange() because there is
        // a possibility that we're not in the main thread, and thus currently
        // blocked in select(). (this functionality is in place for the sake of
        // the HsHa server.)
        state_ = FrameBufferState.AWAITING_REGISTER_WRITE;
      }

	  // 改变secltionKey的感兴趣事件
      requestSelectInterestChange();
    }


    /**
     * When this FrameBuffer needs to change its select interests and execution
     * might not be in its select thread, then this method will make sure the
     * interest change gets done when the select thread wakes back up. When the
     * current thread is this FrameBuffer's select thread, then it just does the
     * interest change immediately.
     */
    private void requestSelectInterestChange() {
      if (Thread.currentThread() == this.selectThread_) {
        changeSelectInterests();
      } else {
		// 利用work线程时,唤醒selectThread
        this.selectThread_.requestSelectInterestChange(this);
      }
    }
{% endcodeblock %}


AbstractSelectThread

{% codeblock AbstractSelectThread-requestSelectInterestChange lang:java %}

    public void requestSelectInterestChange(FrameBuffer frameBuffer) {
      synchronized (selectInterestChanges) {
        selectInterestChanges.add(frameBuffer);
      }
      // wakeup the selector, if it's currently blocked.
      selector.wakeup();
    }


    /**
     * Give this FrameBuffer a chance to set its interest to write, once data
     * has come in.
     */
    public void changeSelectInterests() {
      if (state_ == FrameBufferState.AWAITING_REGISTER_WRITE) {
		// 注册写入事件
        // set the OP_WRITE interest
        selectionKey_.interestOps(SelectionKey.OP_WRITE);
        state_ = FrameBufferState.WRITING;
      } else if (state_ == FrameBufferState.AWAITING_REGISTER_READ) {
		// 准备重新读入，相当于重置为刚创建时的状态
        prepareRead();
      } else if (state_ == FrameBufferState.AWAITING_CLOSE) {
		// 关闭
        close();
        selectionKey_.cancel();
      } else {
        LOGGER.error("changeSelectInterest was called, but state is invalid (" + state_ + ")");
      }
    }


    /**
     * We're done writing, so reset our interest ops and change state
     * accordingly.
     */
    private void prepareRead() {
      // we can set our interest directly without using the queue because
      // we're in the select thread.
      selectionKey_.interestOps(SelectionKey.OP_READ);
      // get ready for another go-around
      buffer_ = ByteBuffer.allocate(4);
      state_ = FrameBufferState.READING_FRAME_SIZE;
    }

{% endcodeblock %}


{% codeblock AbstractSelectThread-执行write方法 lang:java %}

    /**
     * Let a writable client get written, if there's data to be written.
     */
    protected void handleWrite(SelectionKey key) {
      FrameBuffer buffer = (FrameBuffer) key.attachment();
      if (!buffer.write()) {
        cleanupSelectionKey(key);
      }
    }


	// -------FrameBuffer中具体的write输出方法----------------
    /**
     * Give this FrameBuffer a chance to write its output to the final client.
     */
    public boolean write() {
      if (state_ == FrameBufferState.WRITING) {
        try {
		  // 写失败直接返回
          if (trans_.write(buffer_) < 0) {
            return false;
          }
        } catch (IOException e) {
          LOGGER.warn("Got an IOException during write!", e);
          return false;
        }

		// 写完了，重新回到读取状态 
        // we're done writing. now we need to switch back to reading.
        if (buffer_.remaining() == 0) {
          prepareRead();
        }
        return true;
      }

      LOGGER.error("Write was called, but state is invalid (" + state_ + ")");
      return false;
    }

	// ****************  FrameBufferedTransport中 的write方法 ********

	  public void write(byte[] buf, int off, int len) throws TTransportException {
	    writeBuffer_.write(buf, off, len);
	  }
	
	  @Override
	  public void flush() throws TTransportException {
	    byte[] buf = writeBuffer_.get();
	    int len = writeBuffer_.len();
	    writeBuffer_.reset();
		
		// 编码frameSize
	    encodeFrameSize(len, i32buf);
		// 写入4个字节的整数长度
	    transport_.write(i32buf, 0, 4);
		// 写入具体的内容
	    transport_.write(buf, 0, len);
	    transport_.flush();
	  }
	
	  public static final void encodeFrameSize(final int frameSize, final byte[] buf) {
	    buf[0] = (byte)(0xff & (frameSize >> 24));
	    buf[1] = (byte)(0xff & (frameSize >> 16));
	    buf[2] = (byte)(0xff & (frameSize >> 8));
	    buf[3] = (byte)(0xff & (frameSize));
	  }

{% endcodeblock %}


---

TBaseProcessor

{% codeblock TBaseProcessor lang:java %}

	public abstract class TBaseProcessor<I> implements TProcessor {
	  private final I iface;
	  private final Map<String,ProcessFunction<I, ? extends TBase>> processMap;
	
	  protected TBaseProcessor(I iface, Map<String, ProcessFunction<I, ? extends TBase>> processFunctionMap) {
	    this.iface = iface;
	    this.processMap = processFunctionMap;
	  }
	
	  public Map<String,ProcessFunction<I, ? extends TBase>> getProcessMapView() {
	    return Collections.unmodifiableMap(processMap);
	  }
	
	  @Override
	  public boolean process(TProtocol in, TProtocol out) throws TException {
	    TMessage msg = in.readMessageBegin();
		// 获取对应的调用方法
	    ProcessFunction fn = processMap.get(msg.name);
	    if (fn == null) {
	      TProtocolUtil.skip(in, TType.STRUCT);
	      in.readMessageEnd();
	      TApplicationException x = new TApplicationException(TApplicationException.UNKNOWN_METHOD, "Invalid method name: '"+msg.name+"'");
	      out.writeMessageBegin(new TMessage(msg.name, TMessageType.EXCEPTION, msg.seqid));
	      x.write(out);
	      out.writeMessageEnd();
	      out.getTransport().flush();
	      return true;
	    }
	    fn.process(msg.seqid, in, out, iface);
	    return true;
	  }
	}
{% endcodeblock %}

ProcessFunction的处理过程

{% codeblock ProcessFunction lang:java %}

	public abstract class ProcessFunction<I, T extends TBase> {
	  private final String methodName;
	
	  private static final Logger LOGGER = LoggerFactory.getLogger(ProcessFunction.class.getName());
	
	  public ProcessFunction(String methodName) {
	    this.methodName = methodName;
	  }
	
	  public final void process(int seqid, TProtocol iprot, TProtocol oprot, I iface) throws TException {
		// 初始化空参数实例 
	    T args = getEmptyArgsInstance();
	    try {
		  // 读取参数
	      args.read(iprot);
	    } catch (TProtocolException e) {
	      iprot.readMessageEnd();
	      TApplicationException x = new TApplicationException(TApplicationException.PROTOCOL_ERROR, e.getMessage());
	      oprot.writeMessageBegin(new TMessage(getMethodName(), TMessageType.EXCEPTION, seqid));
	      x.write(oprot);
	      oprot.writeMessageEnd();
	      oprot.getTransport().flush();
	      return;
	    }

	    // 读取参数结果 
	    iprot.readMessageEnd();


	    TBase result = null;
	    try {
		  // 调用方法并获取返回结果 
	      result = getResult(iface, args);
	    } catch(Throwable th) {
	      LOGGER.error("Internal error processing " + getMethodName(), th);
	      TApplicationException x = new TApplicationException(TApplicationException.INTERNAL_ERROR, 
	        "Internal error processing " + getMethodName());
	      oprot.writeMessageBegin(new TMessage(getMethodName(), TMessageType.EXCEPTION, seqid));
	      x.write(oprot);
	      oprot.writeMessageEnd();
	      oprot.getTransport().flush();
	      return;
	    }
	
	    if(!isOneway()) {
	      oprot.writeMessageBegin(new TMessage(getMethodName(), TMessageType.REPLY, seqid));
	      result.write(oprot);
	      oprot.writeMessageEnd();
	      oprot.getTransport().flush();
	    }
	  }
	
	  protected abstract boolean isOneway();
	
	  // 获取结果输出的抽象方法
	  public abstract TBase getResult(I iface, T args) throws TException;
	
	  // 参数实例的抽象方法
	  public abstract T getEmptyArgsInstance();
	
	  public String getMethodName() {
	    return methodName;
	  }
	}
{% endcodeblock %}

示例方法调用 

{% codeblock addFolder_result lang:java %}

      public addFolder_result getResult(I iface, addFolder_args args) throws org.apache.thrift.TException {
		// 创建一个result 对象
        addFolder_result result = new addFolder_result();
		// 进行方法调用 
        result.success = iface.addFolder(args.userId, args.name, args.isPublic);

        return result;
      }


	// *********实现方便 协议层 调用 的read 和 write 方法******************

    public void read(org.apache.thrift.protocol.TProtocol iprot) throws org.apache.thrift.TException {
      schemes.get(iprot.getScheme()).getScheme().read(iprot, this);
    }

    public void write(org.apache.thrift.protocol.TProtocol oprot) throws org.apache.thrift.TException {
      schemes.get(oprot.getScheme()).getScheme().write(oprot, this);
      }


	// ******实现了 writeObject 和 readObject 方法***************

    private void writeObject(java.io.ObjectOutputStream out) throws java.io.IOException {
      try {
        write(new org.apache.thrift.protocol.TCompactProtocol(new org.apache.thrift.transport.TIOStreamTransport(out)));
      } catch (org.apache.thrift.TException te) {
        throw new java.io.IOException(te);
      }
    }

    private void readObject(java.io.ObjectInputStream in) throws java.io.IOException, ClassNotFoundException {
      try {
        read(new org.apache.thrift.protocol.TCompactProtocol(new org.apache.thrift.transport.TIOStreamTransport(in)));
      } catch (org.apache.thrift.TException te) {
        throw new java.io.IOException(te);
      }
    }


	// *******************真正实现读写的地方***************

    private static class addFolder_resultStandardScheme extends StandardScheme<addFolder_result> {

      public void read(org.apache.thrift.protocol.TProtocol iprot, addFolder_result struct) throws org.apache.thrift.TException {
        org.apache.thrift.protocol.TField schemeField;
        iprot.readStructBegin();
        while (true)
        {
          schemeField = iprot.readFieldBegin();
          if (schemeField.type == org.apache.thrift.protocol.TType.STOP) { 
            break;
          }
          switch (schemeField.id) {
            case 0: // SUCCESS
              if (schemeField.type == org.apache.thrift.protocol.TType.STRUCT) {
                struct.success = new CodeMsg();
                struct.success.read(iprot);
                struct.setSuccessIsSet(true);
              } else { 
                org.apache.thrift.protocol.TProtocolUtil.skip(iprot, schemeField.type);
              }
              break;
            default:
              org.apache.thrift.protocol.TProtocolUtil.skip(iprot, schemeField.type);
          }
          iprot.readFieldEnd();
        }
        iprot.readStructEnd();

        // check for required fields of primitive type, which can't be checked in the validate method
        struct.validate();
      }

      public void write(org.apache.thrift.protocol.TProtocol oprot, addFolder_result struct) throws org.apache.thrift.TException {
        struct.validate();

        oprot.writeStructBegin(STRUCT_DESC);
        if (struct.success != null) {
          oprot.writeFieldBegin(SUCCESS_FIELD_DESC);
		  // struct.success 是CodeMsg 对象
          struct.success.write(oprot);
          oprot.writeFieldEnd();
        }
        oprot.writeFieldStop();
        oprot.writeStructEnd();
      }

    }

{% endcodeblock %}


再看CodeMsg 对象

{% codeblock CodeMsgStandardScheme lang:java %}

	private static class CodeMsgStandardScheme extends StandardScheme<CodeMsg> {

	    public void read(org.apache.thrift.protocol.TProtocol iprot, CodeMsg struct) throws org.apache.thrift.TException {
	      org.apache.thrift.protocol.TField schemeField;
	      iprot.readStructBegin();
	      while (true)
	      {
	        schemeField = iprot.readFieldBegin();
	        if (schemeField.type == org.apache.thrift.protocol.TType.STOP) { 
	          break;
	        }
	        switch (schemeField.id) {
	          case 1: // CODE
	            if (schemeField.type == org.apache.thrift.protocol.TType.I32) {
	              struct.code = iprot.readI32();
	              struct.setCodeIsSet(true);
	            } else { 
	              org.apache.thrift.protocol.TProtocolUtil.skip(iprot, schemeField.type);
	            }
	            break;
	          case 2: // MSG
	            if (schemeField.type == org.apache.thrift.protocol.TType.STRING) {
	              struct.msg = iprot.readString();
	              struct.setMsgIsSet(true);
	            } else { 
	              org.apache.thrift.protocol.TProtocolUtil.skip(iprot, schemeField.type);
	            }
	            break;
	          default:
	            org.apache.thrift.protocol.TProtocolUtil.skip(iprot, schemeField.type);
	        }
	        iprot.readFieldEnd();
	      }
	      iprot.readStructEnd();
	
	      // check for required fields of primitive type, which can't be checked in the validate method
	      if (!struct.isSetCode()) {
	        throw new org.apache.thrift.protocol.TProtocolException("Required field 'code' was not found in serialized data! Struct: " + toString());
	      }
	      struct.validate();
	    }
	
		// CodeMsg 的写入操作
	    public void write(org.apache.thrift.protocol.TProtocol oprot, CodeMsg struct) throws org.apache.thrift.TException {
	      struct.validate();
		  // 写入结构体名称
	      oprot.writeStructBegin(STRUCT_DESC);
	
		  // 开始写入code 字段名
	      oprot.writeFieldBegin(CODE_FIELD_DESC);
		  // 写入code 字段值
	      oprot.writeI32(struct.code);
		  // 写入一个字段完成
	      oprot.writeFieldEnd();
	
	      if (struct.msg != null) {
	        oprot.writeFieldBegin(MSG_FIELD_DESC);
	        oprot.writeString(struct.msg);
	        oprot.writeFieldEnd();
	      }
		  // 停止字段写入
	      oprot.writeFieldStop();
		  // 写入当前对象完成
	      oprot.writeStructEnd();
	    }

	}

{% endcodeblock %}

