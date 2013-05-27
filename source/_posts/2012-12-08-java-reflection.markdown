---
layout: post
title: "Java 反射"
date: 2012-12-08 15:25
comments: true
categories: Java Reflection
---

Java的反射允许我们在运行时动态地获取类及类成员的类型信息，动态调用类的方法或修改类的成员（Field）。

Java反射相关的类，除了java.lang.Class 以外，大部分都位于java.lang.reflect包下。

Java反射相关的对象：

- Class : 表示Java类相关的类型信息。
- Field : 表示类的成员变量的类型信息。
- Method : 表示类的成员方法的类型信息。
- Constructor : 表示类的构造方法的类型信息。
- Array : 表示数组类型，提供了动态根据目标类型创建数组，以及访问数组元素的静态方法。


####一、获取Class 类型信息

#####1. 通过getClass 获取运行时实例的类型信息
		
		HelloClass h = new HelloClass();
		System.out.println(h.getClass());

		Date d = new Date();
		System.out.println(d.getClass());

#####2. 通过className 加载class类型信息

		String dClassName = "java.util.Date";
		System.out.println(Class.forName(dClassName));

#####3. 通过类型.class 访问对象的类型信息

		System.out.println(Date.class);
		System.out.println(int.class);
		System.out.println(int[].class);
		System.out.println(String[].class);

#####4. 原始类型的Class

一个Class对象实际上表示的是一个类型，而这个类型不一定是一种类。例如，int不是类，但int.class是一个Class类型的对象。

	System.out.println(Integer.class);
	System.out.println(int.class);
	System.out.println("Integer.class.equals(int.class)" + Integer.class.equals(int.class));

	System.out.println(Integer.TYPE);
	System.out.println("Integer.TYPE.equals(int.class) = " + Integer.TYPE.equals(int.class));
		
**结论**：int.class 和 int包装类Interger.class 并不相等，可以通过 Integer.Type 返回原始类型的Class对象。

####二、Class 对象的主要方法

- forName(String className) 返回与带有给定字符串名的类或接口相关联的 Class 对象。
- getAnnotation(Class&lt;A&gt; annotationClass) 如果存在该元素的指定类型的注释，则返回这些注释，否则返回 null。
- getAnnotations() 返回此元素上存在的所有注释。
- getComponentType() 返回表示数组组件类型的 Class。
- getDeclaredConstructor(Class&lt;?&gt;... parameterTypes) 返回一个 Constructor 对象，该对象反映此 Class 对象所表示的类或接口的指定构造方法。
- getDeclaredConstructors() 返回 Constructor 对象的一个数组，这些对象反映此 Class 对象表示的类声明的所有构造方法。
- getDeclaredField(String name) 返回一个 Field 对象，该对象反映此 Class 对象所表示的类或接口的指定已声明字段。
- getDeclaredFields() 返回 Field 对象的一个数组，这些对象反映此 Class 对象所表示的类或接口所声明的所有字段。
- getDeclaredMethod(String name, Class&lt;?&gt;... parameterTypes) 返回一个 Method 对象，该对象反映此 Class 对象所表示的类或接口的指定已声明方法。
- getDeclaredMethods() 返回 Method 对象的一个数组，这些对象反映此 Class 对象表示的类或接口声明的所有方法，包括公共、保护、默认（包）访问和私有方法，但不包括继承的方法。
- getGenericInterfaces() 返回表示某些接口的 Type，这些接口由此对象所表示的类或接口直接实现。
- getGenericSuperclass() 返回表示此 Class 所表示的实体（类、接口、基本类型或 void）的直接超类的 Type。
- getSuperclass() 返回表示此 Class 所表示的实体（类、接口、基本类型或 void）的超类的 Class。
- isAssignableFrom(Class<?> cls) 判定此 Class 对象所表示的类或接口与指定的 Class 参数所表示的类或接口是否相同，或是否是其超类或超接口。
- isInstance(Object obj) 判定指定的 Object 是否与此 Class 所表示的对象赋值兼容。
- newInstance() 创建此 Class 对象所表示的类的一个新实例。

####三、利用Class对象进行动态编程

#####动态建立类的新实例

######1. 通过 Class.newInstance 构建一个对象的新实例

		Date date = Date.class.newInstance();
		System.out.println(date);

Class newInstance 要求这个类有默认的无参数构造器，如果没有默认构造器，就会抛出一个异常。

######2. 通过获取 Constructor，构造带参数的实例

		Constructor<Integer> constructor = Integer.class.getDeclaredConstructor(String.class);
		Integer i =  constructor.newInstance("12");
		System.out.println(i);

######3.动态获取Field的值

先创建一个普通的POJO对象。

	public class Person {
	
		private static int version = 1;
	
		private String name;
	
		private int age;
	
		private boolean sex;
	
		public Person(){};
	
		public Person(String name, int age, boolean sex){
			this.name = name;
			this.age = age;
			this.sex = sex;
		}
	
		public static void sayHi(Person p){
			System.out.println("hello, " + p.getName());
		}
	
		public void introduceSelf(){
			System.out.println("Hi,I'm " + this.name);
		}
	
		... //get set methods 	
	}

获取所有Field的值 

	public class ReflectField {
	
		public static void main(String[] args) throws IllegalArgumentException, IllegalAccessException {
	
			Person p = new Person("Jack",22,true);
	
			Field[] fields = Person.class.getDeclaredFields();
	
			for (Field field : fields) {
				// 如果没有访问权限，则设置访问权限
				if(!field.isAccessible()){
					field.setAccessible(true);
				}
				Object value = field.get(p);
				Class<?> fieldType = field.getType();
	
				System.out.println("field name=" + field.getName()
						+ ",type="+ fieldType.getName()
						+  ",value=" + value);
			}	
		}	
	}


######4.动态调用方法

使用的API:  Method对象的invoke方法  
public Object invoke(Object obj, Object... args)  

- 如果底层方法是静态的，那么可以忽略指定的 obj 参数。该参数可以为 null。   
- 如果底层方法所需的形参数为 0，则所提供的 args 数组长度可以为 0 或 null。   
- 返回值：如果方法正常完成，则将该方法返回的值返回给调用者；如果该值为基本类型，则首先适当地将其包装在对象中。但是，如果该值的类型为一组基本类型，则数组元素不 被包装在对象中；换句话说，将返回基本类型的数组。如果底层方法返回类型为 void，则该调用返回 null。 

<pre>
	public class MethodInvoker {
	
		public static void main(String[] args)throws Exception {
	
			Person jack = new Person("Jack",22,true);
			Person petter = new Person("Petter",22,true);
	
			// 调用类的静态方法
			Method staticMethod = Person.class.getMethod("sayHi", Person.class);
			staticMethod.invoke(null, jack);// 调用静态方法目标对象传null
	
			// 调用类的实例方法
			Method instanceMethod = Person.class.getMethod("introduceSelf", new Class<?>[]{});
			instanceMethod.invoke(petter, new Object[]{});
	
			// 调用泛型方法
			List<String> list = new ArrayList<String>();
			Method genericMethod = list.getClass().getMethod("add", Object.class);
			genericMethod.invoke(list, "hello");
			System.out.println(list.toString());
		}	
	}
</pre>

由于泛型方法的实际类型在编译后被擦除，所以直接使用泛型容器的接口Map.class作为方法参数类型来作获取包含泛型参数的方法。

######5.动态创建数组

使用的API:

- Class.getComponentType() 来返回数组元素的类型
- Array.newInstance(Class&lt;?&gt; componentType, int length)创建一个具有指定的组件类型和长度的新数组。
<pre>
	public class RefkectArray {
	
		public static void main(String[] args) {
			Integer[] intArray = {1,2,3,4,5};
			Number[] numArray = copyOf(intArray, 3, Number[].class);
			System.out.println(Arrays.toString(numArray));
		}
	
	
		// 来自 java.util.Arrays.copyOf
	    @SuppressWarnings("unchecked")
		public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
	
	    	T[] copy = ((Object)newType == (Object)Object[].class)
	            ? (T[]) new Object[newLength]
	            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
	
	        System.arraycopy(original, 0, copy, 0,
	                         Math.min(original.length, newLength));
	        return copy;
	    }
	}
</pre>

####四、Method 对象的主要方法

- getAnnotation(Class&lt;A&gt; annotationClass) 如果存在该元素的指定类型的注释，则返回这些注释，否则返回 null。
- getAnnotations() 返回此元素上存在的所有注释。
- getGenericParameterTypes() 按照声明顺序返回 Type 对象的数组，这些对象描述了此 Method 对象所表示的方法的形参类型的。
- getGenericReturnType()  返回表示由此 Method 对象所表示方法的正式返回类型的 Type 对象。
- getModifiers() 以整数形式返回此 Method 对象所表示方法的 Java 语言修饰符。
- getParameterAnnotations() 返回表示按照声明顺序对此 Method 对象所表示方法的形参进行注释的那个数组的数组。
- getParameterTypes() 按照声明顺序返回 Class 对象的数组，这些对象描述了此 Method 对象所表示的方法的形参类型。
- getReturnType() 返回一个 Class 对象，该对象描述了此 Method 对象所表示的方法的正式返回类型。
- invoke(Object obj, Object... args) 对带有指定参数的指定对象调用由此 Method 对象表示的底层方法。


**Modifier**

Modifier 类提供了 static 方法和常量，对类和成员访问修饰符进行解码。Modifier.toString(int mod) 返回描述指定修饰符中的访问修饰符标志的字符串。

####五、利用反射读取一个类的完整信息（来自Core Java中的示例）

	// 源自Core Java中的 ReflectionTest
	public class ReflectClass {
	
		public static void main(String[] args) throws Throwable {
	
			String name;
	
			if(args.length >0){
				name = args[0];
			}else{
				Scanner in = new Scanner(System.in);
				System.out.println("Please enter the class name(e.g. java.lang.String):");
				name = in.next();
			}
	
			printClass(name);
		}
	
		private static void printClass(String className) throws Throwable{
	
			Class<?> clazz = Class.forName(className);
	
			String modifers = Modifier.toString(clazz.getModifiers());
			if(modifers.length() > 0);
			System.out.print(modifers + " " + clazz.getName() + " ");
	
			Class<?>  superClasses = clazz.getSuperclass();
			if(superClasses != null && !Object.class.equals(superClasses)){
				System.out.print("extends " + superClasses.getName() + " ");
			}
	
			Class<?>[] interfaces =  clazz.getInterfaces();
			if(interfaces.length > 0){
				System.out.print(" implements ");
				for(Class<?> interfacez : interfaces){
					System.out.print(interfacez.getName() + ", ");
				}
			}
			System.out.println();
	
			printFields(clazz);
			printConstructors(clazz);
			printMethods(clazz);
		}
	
		private static void printFields(Class<?> clazz){
			Field[] fields = clazz.getDeclaredFields();
			for(Field field : fields){
	
				String modifers = Modifier.toString(field.getModifiers());
				if(modifers.length() > 0);
				System.out.print(modifers + " ");
	
				System.out.println(field.getType().getName() + " " + field.getName());
			}
		}
	
		private static void printConstructors(Class<?> clazz){
	
			Constructor<?>[] constructors = clazz.getConstructors();
			for(Constructor<?> constructor : constructors){
	
				String modifers = Modifier.toString(constructor.getModifiers());
				if(modifers.length() > 0);
				System.out.print(modifers + " ");
	
				System.out.print(constructor.getName() + "(");
	
				Class<?>[] parameterTypes = constructor.getParameterTypes();
				for(Class<?> parameterType : parameterTypes){
					System.out.print(parameterType.getName() + ", ");
				}
				System.out.println(")");
			}
		}
	
		private static void printMethods(Class<?> clazz){
	
			Method[] methods = clazz.getMethods();
			for(Method method : methods){
	
				String modifers = Modifier.toString(method.getModifiers());
				if(modifers.length() > 0);
				System.out.print(modifers + " ");
	
				Class<?> returnType = method.getReturnType();
				System.out.print(returnType.getName() + " ");
	
				System.out.print(method.getName() + "(");
	
				Class<?>[] parameterTypes = method.getParameterTypes();
				for(Class<?> parameterType : parameterTypes){
					System.out.print(parameterType.getName() + ", ");
				}
				System.out.println(")");
			}
		}
	}


####六、Java中的类型体系

##### Type 接口
位于java.lang.reflect 包内。Type 是 Java 编程语言中所有类型的公共高级接口。它们包括原始类型、参数化类型、数组类型、类型变量和基本类型。 

![](/pics/java-type-interface-uml.jpg)

● Class 类是Type 的直接实现类，描述具体的类型。Class 类的实例表示正在运行的 Java 应用程序中的类和接口。枚举是一种类，注释是一种接口。每个数组属于被映射为 Class 对象的一个类，所有具有相同元素类型和维数的数组都共享该 Class 对象。基本的 Java 类型（boolean、byte、char、short、int、long、float 和 double）和关键字 void 也表示为 Class 对象。 

● ParameterizedType 表示参数化类型，如 Collection<String>。 

1. getActualTypeArguments() 返回表示此类型实际类型参数的 Type 对象的数组。例如Map&lt;Long,String&gt; 将返回 java.lang.Long,java.lang.String.
1. getOwnerType() 返回 Type 对象，表示此类型是其成员之一的类型。
1. getRawType() 返回 Type 对象，表示声明此类型的类或接口。例如Map&lt;Long,String&gt; 将返回 Map.

● TypeVariable 是各种类型变量的公共高级接口。描述类型变量（如 T extends Comparable&lt;? super &gt;）

1. getBounds() 返回表示此类型变量上边界的 Type 对象的数组。
2. getGenericDeclaration() 返回 GenericDeclaration 对象，该对象表示声明此类型变量的一般声明。
3. getName() 返回此类型变量的名称，它出现在源代码中。

● WildcardType 表示一个通配符类型表达式，如 ?、? extends Number 或 ? super Integer。 

1. getLowerBounds() 返回表示此类型变量下边界的 Type 对象的数组。
2. getUpperBounds() 返回表示此类型变量上边界的 Type 对象的数组。

● GenericArrayType 表示一种数组类型，其组件类型为参数化类型或类型变量。

1. getGenericComponentType() 返回表示此数组的组件类型的 Type 对象。


#### Java reflect 包的其它接口

![](/pics/java-reflect-uml.jpg)

● InvocationHandler 是代理实例的调用处理程序 实现的接口。

Object invoke(Object proxy, Method method, Object[] args) 在代理实例上处理方法调用并返回结果。 

● AnnotatedElement表示目前正在此 VM 中运行的程序的一个已注释元素。该接口允许反射性地读取注释。

● AccessibleObject 类是 Field、Method 和 Constructor 对象的基类。它提供了将反射的对象标记为在使用时取消默认 Java 语言访问控制检查的能力。对于公共成员、默认（打包）访问成员、受保护成员和私有成员，在分别使用 Field、Method 或 Constructor 对象来设置或获取字段、调用方法，或者创建和初始化类的新实例的时候，会执行访问检查。 

● Member成员是一种接口，反映有关单个成员（字段或方法）或构造方法的标识信息。 

● GenericDeclaration声明类型变量的所有实体的公共接口。

TypeVariable&lt;?&gt;[] getTypeParameters()  返回声明顺序的 TypeVariable 对象的数组，这些对象表示由此 GenericDeclaration 对象表示的一般声明声明的类型变量。 

####七、通过反射获取运行时的泛型参数

##### 1. 通过反射获取运行时泛型类的实际泛型参数

使用API： this.getClass().getGenericSuperclass()

	class  BaseDAO<T>{

		@SuppressWarnings("unchecked")
		public void save(T obj){
			Type type = this.getClass().getGenericSuperclass();
			if(type instanceof ParameterizedType){
				Type[] actualTypes  =((ParameterizedType)type).getActualTypeArguments();
				System.out.println("save type " + actualTypes[0]);
				System.out.println("save class " + (Class<T>)actualTypes[0]);
				System.out.println("get by obj " + obj.getClass());

			}
		}
	}

	class PersonDAO extends BaseDAO<Person>{}

	public static void main(String[] args) throws Exception {
		PersonDAO personDao = new PersonDAO();
		personDao.save(new Person());
	}

其中，通过 this.getClass().getGenericSuperclass()获取当前正在运行对象超类的泛型类型，如果超类是个参数化类型，那么获取该参数化类型的实际类型信息。

##### 2. 通过反射获取泛型方法的实际参数类型

使用API：method.getGenericParameterTypes()

	public class ReflectGeneric {
	
		static class  BaseDAO<T>{
	
			@SuppressWarnings("unchecked")
			public void save(T obj){
				Type type = this.getClass().getGenericSuperclass();
				if(type instanceof ParameterizedType){
					Type[] actualTypes  =((ParameterizedType)type).getActualTypeArguments();
					System.out.println("save type " + actualTypes[0]);
					System.out.println("save class " + (Class<T>)actualTypes[0]);
					System.out.println("get by obj " + obj.getClass());
	
				}
			}
	
			// 只做示例：没有返回值 
			public void findByIds(Map<Long, String> IdUserIdMap){}
	
			public void findByIds(Long[] ids){}
	
			public void findByIds(T[] objArray){}
	
		}
	
		static class PersonDAO extends BaseDAO<Person>{
	
		}
	
		public static void main(String[] args) throws Exception {
	
			PersonDAO personDao = new PersonDAO();
			personDao.save(new Person());
	
			Method method = PersonDAO.class.getMethod("findByIds", Map.class);
			printGenericParameterTypes(method);
	
			Method[] methods = BaseDAO.class.getDeclaredMethods();
			for(Method item : methods){
				printGenericParameterTypes(item);
			}
		}
	
		private static void printGenericParameterTypes(Method method) {
	
			String modifers = Modifier.toString(method.getModifiers());
			if(modifers.length() > 0);
			System.out.print(modifers + " ");
	
			System.out.print(method.getName() + "(");
			Type[] genericParameterTypes  = method.getGenericParameterTypes();
			for(Type parameterType : genericParameterTypes){
				printType(parameterType );
				System.out.print(",");
			}
			System.out.println(")");
		}
	
		private static void printType(Type parameterType) {
	
			if(parameterType instanceof Class){
				System.out.print(((Class<?>)parameterType).getName());
	
			}else if(parameterType instanceof ParameterizedType){
				ParameterizedType type = (ParameterizedType)parameterType;
	
				System.out.print(((Class<?>)type.getRawType()).getName()   + "<");
				Type[] types  = ((ParameterizedType) parameterType).getActualTypeArguments();
				for(Type type2 : types){
					System.out.print(((Class<?>)type2).getName() + ",");
				}
				System.out.print("> ");
	
			}else if(parameterType instanceof GenericArrayType){
	
				GenericArrayType type = (GenericArrayType)parameterType;
				System.out.print(type.getGenericComponentType() + "[] ");
	
			}else if(parameterType instanceof TypeVariable){
				System.out.print(((TypeVariable<?>) parameterType).getName());
	
			}else if(parameterType instanceof WildcardType){
	
				WildcardType wildType = (WildcardType)parameterType;
				if(wildType.getUpperBounds().length > 0){
					System.out.print("? extends " + Arrays.toString(wildType.getUpperBounds()));
				}else if(wildType.getLowerBounds().length > 0){
					System.out.print("? super " + Arrays.toString(wildType.getLowerBounds()));
				}
			}
		}
	}
	
以上的示例代码只对类型做简单的判断，没有判断多层级的类型嵌套，完整的示例请参考 《Java核心技术》中的12.9.2 章节中的示例代码。


参考资料

- JDK文档
- 《Java核心技术》
- 《Java Reflection in Action》
- [Java编程 的动态性，第 2部分: 引入反射](http://www.ibm.com/developerworks/cn/java/j-dyn0603/)
- [The Reflection API](http://docs.oracle.com/javase/tutorial/reflect/index.html)
- [Java反射经典实例 Java Reflection Cookbook (初级)](http://www.blogjava.net/jialing/archive/2006/08/24/JavaReflectionCookbook1.html)
- [java 泛型 深入](http://www.blogjava.net/fancydeepin/archive/2012/08/25/java_generics.html) 
- [Java 反射机制深入研究](http://lavasoft.blog.51cto.com/62575/43218/)



