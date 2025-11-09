---
title: Java深拷贝的实现方式
date: 2022-09-16 14:52:44
excerpt: 总结了常见的三种java深拷贝的方法
tags: [JAVA,面向对象,深拷贝]
categories: [技术,JAVA]
---

# 前言

​	阅读本文前，首先要清楚地了解JAVA深浅拷贝的含义以及区别，如果没有这个基础的话，还是先了解一下比较好，本文不再赘述。

![](https://s3.bmp.ovh/imgs/2022/09/16/31c1977863592194.png)

（在CSDN上找到的网图）

# 前置知识

为什么我要写这篇博客呢？因为网上的博客实在是太参差不齐了，里面用到了很多我根本不知道的语法知识，所以在这里先科普一下！

## 注解（Annotation）

举个例子吧：`@Data`就是一个注解。

想知道注解是干什么的吗？移步我另一个博客——>[点这儿](https://pcpas.github.io/2022/09/16/Java%E8%AF%AD%E6%B3%95%E4%B9%8B%E6%B3%A8%E8%A7%A3/)。

## 接口（interface）和实现（implement）

移步我另一个博客——>[点这儿](https://pcpas.github.io/2022/09/16/Java%E8%AF%AD%E6%B3%95%E4%B9%8B%E6%8E%A5%E5%8F%A3/)。

## 接口：Cloneable

Cloneable是标记型的接口，它们内部都没有方法和属性，实现 Cloneable来表示该对象能被克隆，能使用Object.clone()方法。如果没有实现 Cloneable的类对象调用clone()就会抛出CloneNotSupportedException。

要让对象可以被克隆，应具备以下2个条件：

- 让该类实现java.lang.Cloneable接口；

- 重写（Override）Object的clone()方法；（如果不重写的话就默认使用最基础的）

  关于为什么一定要重写：在Java中，深拷贝和浅拷贝是一件很重要的事，即使你想要使用浅拷贝，你也必须重写，因为这让别人（和你自己）了解你的真实意图。

## 接口：Serializable

Serializable接口位于java.io.Serializable包中，一般在创建实体类的时候会去实现这个接口，目的是为了序列化。

## 指针：Super





# 具体实现

先看一段代码：

```java
@Data
public class Worker implements Cloneable {

    private String name;
    private Integer age;
    private String gender;

    private EducationInfo educationInfo;

    public Worker(String name, Integer age, String gender, String school, String time) {
        this.name = name;
        this.age = age;
        this.gender = gender;
        this.educationInfo = new EducationInfo(school, time);
    }

    @Override
    public Object clone() {
        try {
            return super.clone();
        } catch (CloneNotSupportedException e) {
            return null;
        }
    }
}
```

```java
@Data
@AllArgsConstructor
public class EducationInfo implements Cloneable*/{

    private String school;

    private String time;

    @Override
    public Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
```

此时，我们实现的是浅拷贝。

实现深拷贝的三种方式：

第一种方式：直接通过 new 对象创建新的对象，通过 new 出来的对象肯定是在内存中重新开辟一块空间存储所以可以实现深拷贝；

第二种方式：通过调用父类clone再进行重新复制，虽然调用父类 Native 修饰的 clone 方法比第一种方式速度快，此步骤如果继承类中有多个引用类型克隆相对麻烦；

//笔者完全不理解第二种方法的原理，但是时间不够了，以后再看。

第三种方式：通过序列化和反序列化使用流进行深拷贝（注意类都要实现 Serializable 接口），因为保存在流中的数据相当于新的，若要实现对象深拷贝，推荐使用此方式。

我们对前面的Worker进行一些改造:

```java
@Data
public class Worker implements Serializable Cloneable{

    private String name;
    private Integer age;
    private String gender;

    private EducationInfo educationInfo;

    public Worker(String name, Integer age, String gender, String school, String time) {
        this.name = name;
        this.age = age;
        this.gender = gender;
        this.educationInfo = new EducationInfo(school, time);
    }
    
//    @Override
//    public Object clone() {
//
//        // 第一种方式：直接通过new对象创建新的对象
        Worker worker = new Worker(
                name, age, gender, educationInfo.getSchool(), educationInfo.getTime()
        );
        return worker;
//
//        // 第二种方式：通过父类clone再进行赋值
//        try {
//            Worker worker = (Worker) super.clone();
//            worker.educationInfo = (EducationInfo) educationInfo.clone();
//            return worker;
//        } catch (CloneNotSupportedException ex) {
//            return null;
//        }
//    }

//         推荐第三种方式：通过序列化和反序列化使用流进行深拷贝，注意类都要实现Serializable接口
    public Worker clone() {

        Worker worker = null;

        try {
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            ObjectOutputStream oos = new ObjectOutputStream(baos);
            oos.writeObject(this);

            // 将流序列化成对象
            ByteArrayInputStream bais = new ByteArrayInputStream(baos.toByteArray());
            ObjectInputStream ois = new ObjectInputStream(bais);
            worker = (Worker) ois.readObject();
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }

        return worker;
    }
}

```



指路：[理解Java对象浅拷贝和深拷贝以及实现深拷贝的三种方式](https://blog.csdn.net/qq_19244927/article/details/115422770)
