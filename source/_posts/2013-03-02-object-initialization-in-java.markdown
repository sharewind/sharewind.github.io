---
layout: post
title: "Java中的对象初始化"
date: 2013-03-02 10:24
comments: true
categories: Java
---

##### 1. 一般类的初始化顺序  

- 加载类
- 设置所有的静态成员为默认值(0,false,0.0 etc.)，引用类型初始化null
- 按照在类声明中出现的次序，依次执行静态成员或静态块的初始化。静态初始化只在Class对象首次加载的时候进行一次。
- new Class() 创建一个实例对象时,首先在堆上为**实例对象**分配足够的存储空间。
- 这块存储空间会清零，自动将**实例对象**的所有基本类型数据都设置成默认值，而引用类型被设置为null.
- 初始化类的实例成员，实例成员间的初始化顺序取决于在类的中声明顺序。
- 执行构造方法

##### 2. 继承结构的初始化顺序
- 先执行父类的静态成员初始化
- 执行子类的静态成员初始化
- 执行父类的实例成员初始化
- 再执行父类的构造函数
- 执行子类的实例成员初始化
- 执行子类的构造函数

code:

	class Who{
	
		public Who(String name){
			System.out.println(name + " is here!");
		}
	
	}
	
	
	
	class Child extends Parent{
	
		Who w3 = new Who("Child instance");
	
		static{
			System.out.println("Child static block 1");
		}
	
		static Who w4 = new Who("Child static");
	
		static{
			System.out.println("Child static block 2");
		}
	
		public Child(){
			System.out.println("Child construct");
		}
	
	}
	
	
	class Parent{
	
		Who w2 = new Who("Parent instance");
	
		static{
			System.out.println("Parent static block 1");
		}
	
		static Who w1 = new Who("Parent static");
	
		static{
			System.out.println("Parent static block 2");
		}
	
		public Parent(){
			System.out.println("Parent construct");
		}
	
	}
	
	public class Test {
		public static void main(String[] args) {
			new Child();
		}
	}

**输出：**

	Parent static block 1
	Parent static is here!
	Parent static block 2
	Child static block 1
	Child static is here!
	Child static block 2
	Parent instance is here!
	Parent construct
	Child instance is here!
	Child construct


##### 3. 类的静态块中，启动一个线程（线程中包含类的静态变量）

	class Foo {
	
		public Foo(int i) {
			System.out.println("foo" + i + " construct ");
		}
	}
	
	class Example {
	
		static {
			System.out.println("static block 1 ");
			// compile error
			// System.out.println("Print " + foo);
	
			new Thread(new Runnable() {
	
				@Override
				public void run() {
					System.out.println("Thread1 ");
					// System.out.println("Thread run whith foo2 " + foo2);
				}
			}).start();
		}
	
	
		static {
			System.out.println("static block 2 ");
	
			new Thread(new Runnable() {
	
				@Override
				public void run() {
					System.out.println("Thread2 run whith foo " + foo);
					// System.out.println("Thread run whith foo2 " + foo2);
				}
			}).start();
	
			try {
				System.out.println("main thread sleep ");
				Thread.sleep(1000 * 5);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	
		static Foo foo = new Foo(1);
	
		final Foo foo2 = new Foo(2);
	
		static {
			System.out.println("static block 3 ");
		}
	
		public Example() {
			System.out.println("Example construct");
		}
	
	}
	
	public class Test2 {
	
		public static void main(String[] args) {
			new Example();
		}
	}

**输出：**

	static block 1 
	static block 2 
	main thread sleep 
	Thread1 
	foo1 construct 
	static block 3 
	foo2 construct 
	Example construct
	Thread2 run whith foo Foo@10c042ab

**结论：**  
1. 可见如果静态块中创建的线程，如果引用类的静态成员，会等到类的静态成员初始化完成后再执行该线程。

#####5. stackoverflow 上的奇葩问题

	import java.util.ArrayList;
	import java.util.List;
	
	public class Test3 {
		public static void main(String[] args) {
			SomeClass.getInstance();
		}
	}
	
	class SomeClass {
	
		private static final SomeClass instance = new SomeClass();
	
		public static SomeClass getInstance() {
			return instance;
		}
	
		static {
			System.out.println("Someclass static init");
		}
		private static String objectName1 = "test1";
		private static String objectName2 = "test2";
	
		@SuppressWarnings("serial")
		private List<SomeObject> list = new ArrayList<SomeObject>() {
			{
				add(new SomeObject(objectName1));
				add(new SomeObject(objectName2));
			}
		};
	
		public SomeClass(){
			System.out.println("some class construct");
			System.out.println("list is " + list);
		}
	}
	
	class SomeObject {
	
		String name;
	
		SomeObject(String name) {
			this.name = name;
			System.out.println("my name is:" + name);
		}
	}

**输出：**

	my name is:null
	my name is:null
	some class construct
	list is [SomeObject@780adb3f, SomeObject@10c042ab]
	Someclass static init

**牛人解答：**
> Static blocks are initialized in order (so you can rely on the ones above in the ones below). By creating an instance of SomeClass as your first static initializer in SomeClass, you're forcing an instance init during the static init phase.

>So the logical order of execution of your code is:

>- Load class SomeClass, all static fields initially defaults (0, null, etc.)
>- Begin static inits
>- First static init creates instance of SomeClass
>- Begin instance inits for SomeClass instance, using current values for static fields (so objectName1 and objectName2 are null)
>- Load SomeObject class, all static fields initially default (you don't have any)
>- Do SomeObject static inits (you don't have any)
>- Create instances of SomeObject using the passed-in null values
>- Continue static inits of SomeClass, setting objectName1 and objectName2

>To make this work as you may expect, simply put the inits for objectName1 and objectName2 above the init for instance.

---
As suggested moving this line:

	private static final SomeClass  instance    = new SomeClass();

after these:

	private  static String objectName1  ="test1";
	private  static String objectName2  ="test2";

should fix the problem.

#####6. 多线程初始化

**类的初始化**阶段是执行类构造器<code>&lt;clinit&gt;()</code>方法的过程。  

- &lt;clinit&gt;()方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块（static{}块）中的语句合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序决定。
- 虚拟机会保证在子类的&lt;clinit&gt;()方法执行之前，父类的&lt;clinit&gt;()已经执行完毕。
- 由于父类的&lt;clinit&gt;()方法先执行，也就意味着父类中定义的静态语句块要优先于子类的变量赋值操作。
- 如果一个类没有静态语句块，也没有对变更的赋值操作，那么编译器可以不为这个类生成&lt;clinit&gt;()。
- 接口中不能使用静态语句块，但仍然有变量初始化的赋值操作，因此接口与类一样会生成&lt;clinit&gt;()方法。
- 虚拟机会操作一个类的&lt;clinit&gt;()方法在多线程环境中被正确的回销和同步。多个线程同时初始化一个对象，那么只会有一个线程去执行这个类的&lt;clinit&gt;()方法。

示例：

	public class DeadLoopDemo{
	
		static{
	
			if(true){
				System.out.println(Thread.currentThread() + "init DeadLoopDemo");
				while (true) {
	
				}
			}
		}
	
		public static void main(String[] args) {
	
			Runnable runable = new Runnable() {
	
				@Override
				public void run() {
					// TODO Auto-generated method stub
					System.out.println(Thread.currentThread() + "start");
					DeadLoopDemo d1 = new DeadLoopDemo();
					System.out.println(Thread.currentThread() + "stop");
	
				}
			};
	
			Thread t1 = new Thread(runable);
			Thread t2 = new Thread(runable);
			t1.start();
			t2.start();
		}
	}


####参考资料  

- [Object Initialization in Java](http://www.artima.com/designtechniques/initializationP.html)
- [java-constructor-order](http://stackoverflow.com/questions/7687030/java-constructor-order)  - Stack Overflow
- Thinking in Java
- 《深入理解Java虚拟机》——周志明著



