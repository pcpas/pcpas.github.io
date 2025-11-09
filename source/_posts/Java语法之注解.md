---
title: Java语法之注解
date: 2022-09-16 15:04:07
excerpt: Java语法之注解
tags: [JAVA,面向对象,注解]
categories: [技术,JAVA]
---

# 什么是注解

Java注解又称Java标注，是在 JDK5 时引入的新特性，注解（也被称为元数据）。

Java注解它提供了一种安全的类似注释的机制，用来将任何的信息或元数据（meta-data）与程序元素（类、方法、成员变量等）进行关联。

Java注解是附加在代码中的一些元信息，用于一些工具在编译、运行时进行解析和使用，起到说明、配置的功能。

# 注解的分类

## 1.标准注解

包括`@Override`、`@Deprecated`、`@SuppressWarnings`等，使用这些注解后编译器就会进行检查。

### 1.1 @Override

如果试图使用 @Override 标记一个实际上并没有覆写父类对应方法的子类方法时，Java 编译器会Warning。

### 1.2 @Deprecated

@Deprecated 用于标明被修饰的类或类成员、类方法已经废弃、过时，不建议使用。

### 1.3 @SuppressWarnings

@SuppressWarnings 用于关闭对类、方法、成员编译时产生的特定警告。

这个注解的使用方法还比较复杂，建议想使用的话可以特意上网搜搜，这里不再赘述！

### 1.4 更多

Java自带了很多注解，但是一般来说在初级开发中使用的较少，如有需要，专门去查看即可。

## 2.元注解

元注解是Java API提供的，是用于修饰注解的注解，通常用在注解的定义上。

常见的元注解及其含义如图所示：

<img src="https://s3.bmp.ovh/imgs/2022/09/16/3c6372b7482496a6.png"  />

## 3.自定义注解

了解不多，以后用到再回来补充吧！反正是很高级的东西！

# Lombok

Lombok项目是一个Java库，它会自动插入编辑器和构建工具中，Lombok提供了一组有用的注解，用来消除Java类中的大量样板代码。比如使用@Data就可以替换数百行代码从而产生干净，简洁且易于维护的Java类。

## @Data

@Data相当于@Getter @Setter @RequiredArgsConstructor @ToString @EqualsAndHashCode这5个注解的合集

添加@Data注解后可以不用再写

- getter,setter方法,
- toString方法
- hashCode方法
- equals方法
  等等

最后链接一个Lombok@Data的官方文档：[点我](https://projectlombok.org/features/Data)



参考：[Java注解最全详解(超级详细)](https://blog.csdn.net/weixin_68320784/article/details/126365775)
