---
layout: post
title: "BTrace 简介"
date: 2012-12-03 16:28
comments: true
categories: Java Btrace
---
BTrace就是一个可以在不改代码、不重启应用的情况下，动态的查看程序运行细节的工具。

官方网站[http://kenai.com/projects/btrace](http://kenai.com/projects/btrace)

下载地址 [http://kenai.com/projects/btrace/downloads/directory/releases/current](http://kenai.com/projects/btrace/downloads/directory/releases/current)

以下示例代码来源于《深入理解Java虚拟机》


<pre>
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

/**
 *
 * btrace pid TracingScript.class
 * 例如：btrace 8056 TracingScript.class
 * @author Caijianfeng
 *
 */
public class BTraceTest {

	public int add(int a,int b){
		int c = a + b;
		return c;
	}

	public static void main(String[] args) throws IOException {
		BTraceTest bTraceTest = new BTraceTest();
		BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(System.in));
		for (int i = 0; i < 10; i++) {
			bufferedReader.readLine();
			int result = bTraceTest.add(i, i+1);
			System.out.println(result);
		}

	}
}
</pre>


Btrace 脚本


<pre>
import static com.sun.btrace.BTraceUtils.jstack;
import static com.sun.btrace.BTraceUtils.println;
import static com.sun.btrace.BTraceUtils.str;
import static com.sun.btrace.BTraceUtils.strcat;

import com.sun.btrace.annotations.BTrace;
import com.sun.btrace.annotations.Kind;
import com.sun.btrace.annotations.Location;
import com.sun.btrace.annotations.OnMethod;
import com.sun.btrace.annotations.Return;
import com.sun.btrace.annotations.Self;

@BTrace
public class TracingScript{

	@OnMethod(
		clazz = "BTraceTest",
		method = "add",
		location = @Location(Kind.RETURN)
	)
	public static void func(@Self BTraceTest instance,int a,int b,@Return int result){
		println("调用堆栈：");
		jstack();
		println(strcat("方法参数A：",str(a)));
		println(strcat("方法参数B：",str(b)));
		println(strcat("方法结果：",str(result)));
	}

}
</pre>


Btrace 在调试resin 等容器里的代码可能由于采用的是 contentClassLoader 的原因而无法代码成功。

代码库： http://code.google.com/p/swtools/source/browse/#git%2Fjava%2Fbtrace-test

###参考资料

- BTrace使用简介 [http://rdc.taobao.com/team/jm/archives/509](http://rdc.taobao.com/team/jm/archives/509) 介绍更多的Btrace的用法，如代码耗时、代码参数、执行代码的哪一行
- 《深入理解Java虚拟机》——周志明著
