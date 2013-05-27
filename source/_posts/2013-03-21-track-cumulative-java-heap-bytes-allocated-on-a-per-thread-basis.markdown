---
layout: post
title: "跟踪JVM在每个线程上分配的堆内存"
date: 2013-03-21 09:38
comments: true
categories: Java JVM
---

JDK 6 update 25 里添加的一个新功能非常有趣，可以按照线程来跟踪（GC堆）内存的分配量。

JMX中，该功能由ThreadMXBean上新增的几个方法提供。详情可见下面例子。  
ThreadMXBean.getThreadAllocatedBytes(long threadId)的用法基本上可以看成跟System.currentTimeMillis()用于计时的用法一样，在两点上记录并且求差即可。 


	import java.lang.management.ManagementFactory;
	import java.text.DecimalFormat;
	import java.util.ArrayList;
	import java.util.List;
	
	import com.sun.management.ThreadMXBean;
	
	@SuppressWarnings("restriction")
	public class TestThreadAllocateBytes {
	
		public static void main(String[] args) {
	
			for (int i = 0; i < 2; i++) {
				new Thread(new Task()).start();
			}
	
			while(true){
	
			}
		}
	
		public static class Task implements Runnable{
	
			@Override
			public void run() {
	
				ThreadMXBean threadMXBean = (ThreadMXBean) ManagementFactory.getThreadMXBean();
				long startSize = threadMXBean.getThreadAllocatedBytes(Thread.currentThread().getId());
				System.out.println(Thread.currentThread().getName() + " start " + startSize);
	
				String msg = "hello world";
				List<String> list = new ArrayList<>();
				for (int i = 0; i < 10000; i++) {
					list.add(msg + i);
				}
	
				long endSize = threadMXBean.getThreadAllocatedBytes(Thread.currentThread().getId());
				long costBytes = endSize - startSize;
	
				System.out.println(Thread.currentThread().getName() + " end " + endSize);
				System.out.println(Thread.currentThread().getName() + " 消耗了内存： " + readableFileSize(costBytes));
	
	
				try {
					Thread.sleep(1000 * 1000 * 1000);
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
	
		}
	
	
		public static String readableFileSize(long size) {
		    if(size <= 0) return "0";
		    final String[] units = new String[] { "B", "KB", "MB", "GB", "TB" };
		    int digitGroups = (int) (Math.log10(size)/Math.log10(1024));
		    return new DecimalFormat("#,##0.#").format(size/Math.pow(1024, digitGroups)) + " " + units[digitGroups];
		}
	
	}


输出

	Thread-0 start 7568
	Thread-0 end 1862032
	Thread-1 start 48
	Thread-1 end 1849304
	Thread-1 消耗了内存： 1.8 MB
	Thread-0 消耗了内存： 1.8 MB


#### 通过jvisualvm 查看JVM的内存分配

##### 1.查看每个对象占用的堆内存大小

![](/pics/jvisualvm-memory-1.jpg)

#####2.查看每个线程占用的堆内存大小

![](/pics/jvisualvm-memory-2.jpg)



参考资料

- [JDK6u25里添加的按线程统计分配内存量: JMX](http://rednaxelafx.iteye.com/blog/1021619)
- [Monitoring memory allocation per thread](http://blog.jruby.org/2011/12/monitoring-memory_allocation-per-thread/)  
- [Analyze/visualize GC usage patterns between two versions of a program? - Stack Overflow](http://stackoverflow.com/questions/10037723/analyze-visualize-gc-usage-patterns-between-two-versions-of-a-program) 