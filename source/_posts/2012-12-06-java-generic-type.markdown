---
layout: post
title: "Java泛型通配符extends与super"
date: 2012-12-06 21:42
comments: true
categories: Java
---
关键字说明

● ? 通配符类型

●  <? extends T> 表示类型的上界，表示参数化类型的可能是T 或是 T的子类

●  <? super T> 表示类型下界（Java Core中叫超类型限定），表示参数化类型是此类型的超类型（父类型），直至Object

extends 示例

    static class Food{}
    static class Fruit extends Food{}
    static class Apple extends Fruit{}
    static class RedApple extends Apple{}

    List<? extends Fruit> flist = new ArrayList<Apple>();
    // complie error:
    // flist.add(new Apple());
    // flist.add(new Fruit());
    // flist.add(new Object());
    flist.add(null); // only work for null 

List<? extends Frut> 表示 “具有任何从Fruit继承类型的列表”，编译器无法确定List所持有的类型，所以无法安全的向其中添加对象。可以添加null,因为null 可以表示任何类型。所以List 的add 方法不能添加任何有意义的元素，但是可以接受现有的子类型List<Apple> 赋值。
	
	Fruit fruit = flist.get(0);
	Apple apple = (Apple)flist.get(0);

由于，其中放置是从Fruit中继承的类型，所以可以安全地取出Fruit类型。

	flist.contains(new Fruit());	
	flist.contains(new Apple());

在使用Collection中的contains 方法时，接受Object 参数类型，可以不涉及任何通配符，编译器也允许这么调用。

super 示例

	List<? super Fruit> flist = new ArrayList<Fruit>();
	flist.add(new Fruit());
	flist.add(new Apple());
	flist.add(new RedApple());
	// compile error:
	List<? super Fruit> flist = new ArrayList<Apple>();

List<? super Fruit> 表示“具有任何Fruit超类型的列表”，列表的类型至少是一个 Fruit 类型，因此可以安全的向其中添加Fruit 及其子类型。由于List<? super Fruit>中的类型可能是任何Fruit 的超类型，无法赋值为Fruit的子类型Apple的List<Apple>.

	// compile error:
	Fruit item = flist.get(0);

因为，List<? super Fruit>中的类型可能是任何Fruit 的超类型，所以编译器无法确定get返回的对象类型是Fruit,还是Fruit的父类Food 或 Object.

小结

extends 可用于的返回类型限定，不能用于参数类型限定。

super 可用于参数类型限定，不能用于返回类型限定。

> 直观地讲，带有super超类型限定的通配符可以向泛型对象写入，带有extends子类型限定的通配符可以向泛型对象读取。——《Core Java》