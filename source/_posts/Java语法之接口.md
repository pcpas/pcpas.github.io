---
title: Java语法之接口
date: 2022-09-16 15:39:35
excerpt: Java语法之接口
categories: [技术, Java]
tags: [接口, 面向对象]
---

# 接口是什么

Java接口（interface）是一系列方法的声明，是一些方法特征的集合，一个接口只有方法的特征没有方法的实现，因此这些方法可以在不同的地方被不同的类实现，而这些实现可以具有不同的行为（功能）。

# 接口的特点

1. 接口中的每个方法都是隐式抽象的，接口中的方法会被隐式的指定为public abstract
2. 接口中可以含有变量，但是接口中的变量会被隐式的指定为public static [final](https://blog.csdn.net/jiahao1186/article/details/121888509)变量
3. 接口中的方法是不能在接口中实现的，只能由实现接口的类来实现接口中的方法

# 接口和类的区别

1. 接口不能实例化
2. 接口没有构造方法
3. 接口中所有的方法必须是抽象方法，Java8之后可以使用[default](https://blog.csdn.net/lilian9215/article/details/119721637)关键字修饰非抽象方法
4. 接口不能包含成员变量，除了static和final变量
5. 接口支持多实现（相当于C++中的多重继承）

# 接口和抽象类的区别

1. 抽象类中的方法可以有方法体，就是能实现方法的具体功能，但是接口中的方法不能有方法体
2. 抽象类中的成员变量可以使各种类型的，而接口中的成员变量只能是public static final类型的。
3. 接口中不能含有静态代码块以及静态方法，而抽象类中可以有静态代码块和静态方法。
4. 一个类只能继承一个抽象类，而一个类可以实现多个接口。

# 接口的声明

```java
public interface MyComparable{
 
        int comparTo(Object other);

```

使用intrface关键字。

# 接口的实现：implement

extends 继承类；implements 实现接口。 

相当于对于某个接口的实现了。
