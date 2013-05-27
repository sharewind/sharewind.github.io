---
layout: post
title: "java vm args"
date: 2012-12-24 15:42
comments: true
categories: Java JVM
---


#### 1.java 标准参数
<pre>
C:\Users\Administrator>java -h
用法: java [-options] class [args...]
           (执行类)
   或  java [-options] -jar jarfile [args...]
           (执行 jar 文件)
其中选项包括:
    -d32          使用 32 位数据模型 (如果可用)
    -d64          使用 64 位数据模型 (如果可用)
    -server       选择 "server" VM
    -hotspot      是 "server" VM 的同义词 [已过时]
                  默认 VM 是 server.

    -cp <目录和 zip/jar 文件的类搜索路径>
    -classpath <目录和 zip/jar 文件的类搜索路径>
                  用 ; 分隔的目录, JAR 档案
                  和 ZIP 档案列表, 用于搜索类文件。
    -D<name>=<value>
                  设置系统属性
    -verbose[:class|gc|jni]
                  启用详细输出
    -version      输出产品版本并退出
    -version:<value>
                  需要指定的版本才能运行
    -showversion  输出产品版本并继续
    -jre-restrict-search | -no-jre-restrict-search
                  在版本搜索中包括/排除用户专用 JRE
    -? -help      输出此帮助消息
    -X            输出非标准选项的帮助
    -ea[:<packagename>...|:<classname>]
    -enableassertions[:<packagename>...|:<classname>]
                  按指定的粒度启用断言
    -da[:<packagename>...|:<classname>]
    -disableassertions[:<packagename>...|:<classname>]
                  禁用具有指定粒度的断言
    -esa | -enablesystemassertions
                  启用系统断言
    -dsa | -disablesystemassertions
                  禁用系统断言
    -agentlib:<libname>[=<options>]
                  加载本机代理库 <libname>, 例如 -agentlib:hprof
                  另请参阅 -agentlib:jdwp=help 和 -agentlib:hprof=help
    -agentpath:<pathname>[=<options>]
                  按完整路径名加载本机代理库
    -javaagent:<jarpath>[=<options>]
                  加载 Java 编程语言代理, 请参阅 java.lang.instrument
    -splash:<imagepath>
                  使用指定的图像显示启动屏幕
有关详细信息, 请参阅 http://www.oracle.com/technetwork/java/javase/documentation/index.html。

#### 2. java 非标准参数

C:\Users\Administrator>java -X
    -Xmixed           混合模式执行 (默认)
    -Xint             仅解释模式执行
    -Xbootclasspath:<用 ; 分隔的目录和 zip/jar 文件>
                      设置搜索路径以引导类和资源
    -Xbootclasspath/a:<用 ; 分隔的目录和 zip/jar 文件>
                      附加在引导类路径末尾
    -Xbootclasspath/p:<用 ; 分隔的目录和 zip/jar 文件>
                      置于引导类路径之前
    -Xdiag            显示附加诊断消息
    -Xnoclassgc       禁用类垃圾收集
    -Xincgc           启用增量垃圾收集
    -Xloggc:<file>    将 GC 状态记录在文件中 (带时间戳)
    -Xbatch           禁用后台编译
    -Xms<size>        设置初始 Java 堆大小
    -Xmx<size>        设置最大 Java 堆大小
    -Xss<size>        设置 Java 线程堆栈大小
    -Xprof            输出 cpu 配置文件数据
    -Xfuture          启用最严格的检查, 预期将来的默认值
    -Xrs              减少 Java/VM 对操作系统信号的使用 (请参阅文档)
    -Xcheck:jni       对 JNI 函数执行其他检查
    -Xshare:off       不尝试使用共享类数据
    -Xshare:auto      在可能的情况下使用共享类数据 (默认)
    -Xshare:on        要求使用共享类数据, 否则将失败。
    -XshowSettings    显示所有设置并继续
    -XshowSettings:all
                      显示所有设置并继续
    -XshowSettings:vm 显示所有与 vm 相关的设置并继续
    -XshowSettings:properties
                      显示所有属性设置并继续
    -XshowSettings:locale
                      显示所有与区域设置相关的设置并继续

-X 选项是非标准选项, 如有更改, 恕不另行通知。

</pre>


#### 3. java参数与默认值
	使用 -XX:+PrintFlagsFinal 参数可以输出所有参数的名称及默认值。  
	java -XX:+PrintFlagsFinal

#### 4. JVM 常用的参数、Flags 

参数：  
  
    -server       选择 "server" VM
    -cp <目录和 zip/jar 文件的类搜索路径>
    -classpath <目录和 zip/jar 文件的类搜索路径>
                  用 ; 分隔的目录, JAR 档案
                  和 ZIP 档案列表, 用于搜索类文件。
    -D<name>=<value>
                  设置系统属性
    -verbose[:class|gc|jni]
                  启用详细输出
    -version      输出产品版本并退出
    -agentlib:<libname>[=<options>]
                  加载本机代理库 <libname>, 例如 -agentlib:hprof
                  另请参阅 -agentlib:jdwp=help 和 -agentlib:hprof=help
    -agentpath:<pathname>[=<options>]
                  按完整路径名加载本机代理库
    -javaagent:<jarpath>[=<options>]
                  加载 Java 编程语言代理, 请参阅 java.lang.instrument

    -Xloggc:<file>    将 GC 状态记录在文件中 (带时间戳)
    -Xms<size>        设置初始 Java 堆大小
    -Xmx<size>        设置最大 Java 堆大小
	-Xmn<size>        设置Java堆 新生代大小
    -Xss<size>        设置 Java 线程堆栈大小 (对应Flag:ThreadStackSize 默认1MB JDK1.5+)
	-Xnoclassgc       禁用类垃圾收集

Flags:
 	
	-XX:PermSize 指定永久代大小
	-XX:MaxPermSize 指定最大永久代大小
	-XX:SurvivorRatio 新生代中Eden与Survivor 的比例

	-XX:GCTimeRatio   GC时间占总时间的比率，仅使用Parallel Scavenge 收集器时有效
	-Xnoclassgc       禁用类垃圾收集
	
	-XX:+DisableExplicitGC 忽略来自System.gc()方法触发的垃圾收集。
	-XX:+UseParNewGC	  
	-XX:+UseConcMarkSweepGC
	-XX:+CMSPermGenSweepingEnabled
	-XX:+UseCMSCompactAtFullCollection
	-XX:CMSFullGCsBeforeCompaction=0
	-XX:+CMSClassUnloadingEnabled
	-XX:-CMSParallelRemarkEnabled
	-XX:CMSInitiatingOccupancyFraction=70
	-XX:SoftRefLRUPolicyMSPerMB=0

	-XX:+PrintClassHistogram
	-XX:+PrintGCDetails
	-XX:+PrintGCTimeStamps
	-XX:+PrintGCApplicationConcurrentTime
	-XX:+PrintGCApplicationStoppedTime
	-Xloggc:/opt/logs/<services>/gc.log

	调试选项
	-XX:+HeapDumpOnOutofMemoryError	
	-XX:+PrintFlagsFinal 

#### 5. 一个标准的8核CPU的JVM服务配置

	* 内存配置为2g，web服务推荐为4G，RMI服务如果没有大量数据缓存，推荐2G以上
	* gc日志必须打开

	-server
	-Xms2048M
	-Xmx2048M
	-Xmn512M
	-XX:PermSize=256M
	-XX:MaxPermSize=256M
	-XX:SurvivorRatio=8
	-XX:MaxTenuringThreshold=7
	-Xss1m
	-XX:GCTimeRatio=19
	-Xnoclassgc
	-XX:+DisableExplicitGC
	-XX:+UseParNewGC
	-XX:+UseConcMarkSweepGC
	-XX:+CMSPermGenSweepingEnabled
	-XX:+UseCMSCompactAtFullCollection
	-XX:CMSFullGCsBeforeCompaction=0
	-XX:+CMSClassUnloadingEnabled
	-XX:-CMSParallelRemarkEnabled
	-XX:CMSInitiatingOccupancyFraction=70
	-XX:SoftRefLRUPolicyMSPerMB=0
	-XX:+PrintClassHistogram
	-XX:+PrintGCDetails
	-XX:+PrintGCTimeStamps
	-XX:+PrintGCApplicationConcurrentTime
	-XX:+PrintGCApplicationStoppedTime
	-Xloggc:/opt/logs/<services>/gc.log

**高级JVM参数**  

	-XX:HeapDumpPath=/tmp/dis-search.hprof  
	-XX:+HeapDumpOnOutOfMemoryError
####**参考资料**
- [java - the Java application launcher](http://docs.oracle.com/javase/1.4.2/docs/tooldocs/solaris/java.html)
- [Java HotSpot VM Options](http://www.oracle.com/technetwork/java/javase/tech/vmoptions-jsp-140102.html)


