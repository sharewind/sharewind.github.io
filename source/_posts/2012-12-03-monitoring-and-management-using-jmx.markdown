---
layout: post
title: "使用JMX管理和监控JVM"
date: 2012-12-03 17:22
comments: true
categories: Java JVM 
---

JMX（Java Management Extensions，即Java管理扩展）是一个为应用程序、设备、系统等植入管理功能的框架。JMX可以跨越一系列异构操作系统平台、系统体系结构和网络传输协议，灵活的开发无缝集成的系统、网络和服务管理应用。

### 配置jmx
#### 1. 用户名密码的方式 

复制模板文件
<pre>
cp $JAVA_HOME/jre/lib/management/jmxremote.password.template /opt/conf/jmxreomte/jmxremote.password
</pre>

配置示例
<pre>
# The "monitorRole" role has password "QED".
# The "controlRole" role has password "R&D".
monitorRole QED
controlRole R&D
</pre>

权限配置
<pre>
chown nobody.nobody jmxremote.password
chmod 600 jmxremote.password
</pre>
其中，chmod 600 jmxremote.password 等同于 chmod u=rw,g=-,o=- jmxremote.password

#### resin中的配置

###### 1. 默认开启jmxremote 

	<cluster-default>
		<!-- sets the content root for the cluster, relative to server.root -->
		<root-directory>.</root-directory>

		<server-default>
		<!--
			- The JVM arguments
		-->
			<jvm-arg>-d64</jvm-arg>
			<jvm-arg>-Xmx4096m</jvm-arg>
			<jvm-arg>-Xms4096m</jvm-arg>
			<jvm-arg>-Xmn1536m</jvm-arg>
			<jvm-arg>-Xss2m</jvm-arg>
			<jvm-arg>-Xdebug</jvm-arg>
			<jvm-arg>-XX:MaxPermSize=512m</jvm-arg>
			<jvm-arg>-Dcom.sun.management.jmxremote</jvm-arg>
			<jvm-arg>-Dcom.sun.management.jmxremote.ssl=false</jvm-arg>
			<jvm-arg>-Dcom.sun.management.jmxremote.authenticate=true</jvm-arg>


###### 2.具体实例监听的IP与端口

	<cluster id="cluster-01">
		<server id="server-01" address="127.0.0.1" port="6801">
			<http address="192.168.1.10" port="8081" />
			<jvm-arg>-Djava.rmi.server.hostname=192.168.1.10</jvm-arg>
			<jvm-arg>-Dcom.sun.management.jmxremote.port=9081</jvm-arg>
			<jvm-arg>-Dcom.sun.management.jmxremote.password.file=/opt/conf/jmxremote/jmxremote.password</jvm-arg>
		</server>

#####3. 命令行配置
	
	JMXREMOTE=" -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Djava.rmi.server.hostname=192.168.1.10 -Dcom.sun.management.jmxremote.port=9081 "
	java ${JMXREMOTE} Main


#### 参考资料
- [Monitoring and Management Using JMX](http://docs.oracle.com/javase/1.5.0/docs/guide/management/agent.html)
- [ Monitoring and Management Properties](http://docs.oracle.com/javase/6/docs/technotes/guides/management/agent.html#gdeum) - jmxremote 的属性及默认值
- [百度百科-JMX ](http://baike.baidu.com/view/866268.htm)

