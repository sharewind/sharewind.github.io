---
layout: post
title: "Java Profile"
date: 2012-12-21 00:07
comments: true
categories: Java JVM
---

通常进行系统问题定位的时候，可以通过一些数据进行分析。数据包括：运行日志、异常堆栈、GC日志、线程的快照（threaddump/javacore文件）、堆转存快照(heapdump/hprof文件)等。

这里简单介绍一下Java性能诊断的常规工具。

1.jps (JVM Process Status Tool),显示指定系统内所有的Hotspot虚拟机进程。  
	用法：jsp -v

	-m 输出虚拟进程启动时传递给主类main()函数的参数  
	-l 输出主类全名，如果进行执行的是jar包，输出jar路径  
	-v 输出虚拟机进程启动时JVM参数

2.jstat: 观察GC情况  

	jstat -gc pid 2000  
	jstat -gcutil pid 2000

	[jstat - Java Virtual Machine Statistics Monitoring Tool](http://docs.oracle.com/javase/1.5.0/docs/tooldocs/share/jstat.html)

	或者使用top, ps aux 查看总的内存占用情况

3.jinfo 查看与调整虚拟机的各项参数

<pre>
	jinfo -h  
	Usage:  
	    jinfo [option] <pid>  
	        (to connect to running process)  
	    jinfo [option] <executable <core>  
	        (to connect to a core file)  
	    jinfo [option] [server_id@]<remote server IP or hostname>  
	        (to connect to remote debug server) 
	
	where <option> is one of:  
	    -flag <name>         to print the value of the named VM flag  
	    -flag [+|-]<name>    to enable or disable the named VM flag  
	    -flag <name>=<value> to set the named VM flag to the given value  
	    -flags               to print VM flags  
	    -sysprops            to print Java system properties  
	    <no option>          to print both of the above  
	    -h | -help           to print this help message
</pre>


4.jmap ,查看 heap 情况，如查看存活对象列表

	jmap -histo:live pid|grep com.company|less

	或者dump 内存来分析

	jmap -dump:file=test.bin pid

	jmap 查看Java堆的详细信息

	jmap -heap pid	

5.分析dump 堆文件，可以用jhat:
	
	jhat test.bin
	
	分析完成后可以用浏览器查看堆的情况。
	还可以用 Eclipse Memory Analyzer (MAT)工作进行分析，或者IBM的Heap Analyzer.

6.jstack : Java堆栈跟踪工具
	
	jstack pid > thread_dump 

	jstack -l 除了堆栈以外显示关于锁的附加信息

7.jvisualvm 和 jconsole, JVM 自带的图形化工具

8.Btrace 

####参考资料

- 《深入理解Java虚拟机》- 周志明著 第4章
- [Java程序员常用工具集 - 庄周梦蝶](http://www.blogjava.net/killme2008/archive/2012/04/17/374936.html)
- [Java 6 JVM参数选项大全（中文版）](http://kenwublog.com/docs/java6-jvm-options-chinese-edition.htm)
- [JDK Tools and Utilities](http://docs.oracle.com/javase/7/docs/technotes/tools/index.html)
- [Command-Line Options - Troubleshooting Guide for HotSpot VM](http://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/clopts.html)
- [JDK Troubleshooting Guide](http://docs.oracle.com/javase/7/docs/webnotes/tsg/index.html)
- [Java HotSpot VM Options](http://www.oracle.com/technetwork/java/javase/tech/vmoptions-jsp-140102.html)