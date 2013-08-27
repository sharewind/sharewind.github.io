---
layout: post
title: "figure out how logger get the source method name"
date: 2013-08-27 15:26
comments: true
categories: Java
---

{% codeblock info in Logger lang:java %}

	package java.util.logging;
	
	public class Logger {
		
	    public void info(String msg) {
	        if (Level.INFO.intValue() < levelValue) {
	            return;
	        }
	        log(Level.INFO, msg);
   		}

	    /**
	     * Log a message, with no arguments.
	     * <p>
	     * If the logger is currently enabled for the given message
	     * level then the given message is forwarded to all the
	     * registered output Handler objects.
	     * <p>
	     * @param   level   One of the message level identifiers, e.g., SEVERE
	     * @param   msg     The string message (or a key in the message catalog)
	     */
	    public void log(Level level, String msg) {
	        if (level.intValue() < levelValue || levelValue == offValue) {
	            return;
	        }
	        LogRecord lr = new LogRecord(level, msg);
	        doLog(lr);
	    }

{% endcodeblock %}



{% codeblock LogRecord lang:java %}

    public LogRecord(Level level, String msg) {
        // Make sure level isn't null, by calling random method.
        level.getClass();
        this.level = level;
        message = msg;
        // Assign a thread ID and a unique sequence number.
        sequenceNumber = globalSequenceNumber.getAndIncrement();
        threadID = defaultThreadID();
        millis = System.currentTimeMillis();
        needToInferCaller = true;
   	}


    public String getSourceClassName() {
        if (needToInferCaller) {
            inferCaller();
        }
        return sourceClassName;
    }

	// 这里就是找出当前执行的类与方法的地方
    // Private method to infer the caller's class and method names
    private void inferCaller() {
        needToInferCaller = false;
        JavaLangAccess access = SharedSecrets.getJavaLangAccess();
        Throwable throwable = new Throwable();
        int depth = access.getStackTraceDepth(throwable);

        String logClassName = "java.util.logging.Logger";
        String plogClassName = "sun.util.logging.PlatformLogger";
        boolean lookingForLogger = true;
        for (int ix = 0; ix < depth; ix++) {
            // Calling getStackTraceElement directly prevents the VM
            // from paying the cost of building the entire stack frame.
            StackTraceElement frame =
                access.getStackTraceElement(throwable, ix);
            String cname = frame.getClassName();
            if (lookingForLogger) {
				// 跳过所有looger之前的栈帧
                // Skip all frames until we have found the first logger frame.
                if (cname.equals(logClassName) || cname.startsWith(plogClassName)) {
                    lookingForLogger = false;
                }
            } else {
                if (!cname.equals(logClassName) && !cname.startsWith(plogClassName)) {
					// 跳过反射调用
                    // skip reflection call
                    if (!cname.startsWith("java.lang.reflect.") && !cname.startsWith("sun.reflect.")) {
                       // We've found the relevant frame.
                       setSourceClassName(cname);
                       setSourceMethodName(frame.getMethodName());
                       return;
                    }
                }
            }
        }
        // We haven't found a suitable frame, so just punt.  This is
        // OK as we are only committed to making a "best effort" here.
    }

	

{% endcodeblock %}


