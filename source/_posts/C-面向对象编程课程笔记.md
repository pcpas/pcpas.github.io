---
title: C++ 面向对象编程课程笔记
excerpt: 虽然只是门通识课.但是却是buaa难得的好课
date: 2022-05-06 22:02:32
categories: [技术,C++]
tags: [C++,面向对象,课程笔记]
---
# C++与面向对象编程

## 1. A better C

### 1.1 Macro(宏)

#### 1.1.1 常量宏

#define PI 3.14  //no magic number

#### 1.1.2 函数宏

为了避免反复调用小函数(因为调用函数会有额外开销)

#define ARRAY_SIZE(a) sizeof(a)/sizeof(a[0])

#define ADD(a,b)  a+b

//但是实际不好用,非常容易造成误读,不要写

#### 1.1.3 控制宏(开关)

```c++
#define LOCAL_VER

#ifdef LOCAL_VER
	cout<<"connect ximenzi"<<endl;
#else
	cout<<"connect other type"<<endl;
#endif
	return 0;
```

 头文件里有什么?

\\\macro

\\\function declearation 函数声明

    void fun();

\\\struct

```
#ifndef MY_H
#define MY_H
//每个头文件都应当包含,为了防止结构重复声明
//坚决不允许不同头文件的防重引用宏重名
struct Node
{
	int data;
	Node* next
}
```

(如果头文件被包含多次, 那么会引起重复声明redeclearation的问题)

//补充内容:

    a.c   -->   a.o

    b.c   -->   b.o   -->    ab.exe

    compiler     linker

C++ 引用 C函数

```c++
extern "C" void c_fun();
int main()
{
	c_fun();
	return 0;
}
//头文件中通用写法
#ifdef __cplusplus
extern "C"
{
#endif
	void c_fun();
#ifdef __cplusplus
}
#endif

```

### 1.2 C++小知识点

#### 1.2.1 About function  in CPP

##### 1.2.1.1 Overloading 重载

```
//1.同名不同参
void Print(int i)
{
}
void Print(char *str)
{
}

```

##### 1.2.1.2 Default parameter

```
//可以只传2个也可以传3个
//当只有一个默认参数时,必须是最后一个
//最好把最不易变的放在最后面
//声明里有,函数体里就不能有
void f(int a,int b,int c=5)
{
}
int main()
{
	f(1,2,3);
	return 0;
}
```

##### 1.2.1.3 占位符

```
//几乎没用
//一般只会出现在 同名函数,同参,但是含义不同,可以强制不同参做一个区分
void f3(int)
{
}
int main()
{
	f(1);
	return 0;
}
```

#### 1.2.2 引用(reference)

引用 ——**一种安全的指针**

```
int main()
{
	int *pl;
	int a = 10;
	int& r = a;//引用被强制必须初始化 
	//引用是常量,只有初始化一个时候能赋值
	r++;//r++ == a++ r代表a的值
	//那为什么说r是指针?
	fun(a);
	cout<< a <<endl;//输出11
	return 0;
}
void fun(int& m)//借值之名,行指针之实
{
	m++;
}
//指针是丑陋的,但是简明
//引用是简洁的,但是含糊不清
```

#### 1.2.3 堆、栈和内存

```
//memory
//代码区 占多少取决于文件有多大 (一般PC级以上不关心这个问题,不缺这点空间)
//全局变量区 
int g_i;//不需要初始化,因为会被初始清空(全局变量初始化在main函数开始之前)
//运行内存(Runtime Memory)
//   stack    vs     heap(dynamic memory arrange)
// local var           malloc new
// 连续内存(快)         离散空间(有寻址机制)
//					内存的碎片化(缺少连续的大的内存)-->因而产生了极端的内存管理机制
int main()
{
	int *p=&g_i;
	cout<< *(p+100) << endl;//输出0
	//证明全局变量内存会被清空一大片!!这是为程序员提前腾出很多空间
	//但是局部变量不会自己初始化,因为此时程序已经开始运行,要追求效率
	return 0;
}
```

## 2. 封装

### 2.1 Everything is object(面向对象)

封装（1个类）、继承（类与类）、多态（类行为）

面向过程（以事为中心） 面向对象（C++、Java、Python）（物中心）

C语言时代的面向对象 struct ----> 仅仅是相关数据的集合,与类型相关的操作还是独立于类型之外

C++ : class -> 包含属性(attribute)和方法(method)->解决命名危机

类是唯一的抽象 , 对象是无穷多的实例.

类需要实例化成为对象

小知识: 我们关注的内存是runtime-memory, 不关心代码本身的大小

```
class Student
{
	int Id;
	string name;
	void initialStdu(int aid,char *aname);
}
void Student::initialStud(int aid,char *aname)
{
	this->Id = aid;
	this->name = aname; // the hiden this pointer
	//等价于 name = aname;
	// this == 调用者的地址
	// this == &s
}
```

```
#pragma pack(1)
//按1字节对齐,节省空间,便于运输,一般写在头文件
//但是会耽误性能
struct Test
{
	int i;//4
	double d;//8
	char c;//1
	//这个顺序就是24字节
	int i;//4
	char c;//1
	double d;//8
	//这个顺序就是16字节
	void fun();
};
void Test::fun();
//对于sizeof没有影响,因为对于运行内存没有影响;

//分配内存规则: 按大对齐,按大分配
int main()
{
	cout << sizeof(Test) << endl;
	//sizeof()是一种运算符
	return 0;
}

//结论: 类的大小只取决于属性
```

```
struct Test
{
	int a:1;//大小为4字节
	int b:1;
	// 含义: a占1比特 (位结构)
	// 含义: 由a,b以及剩下的30个字母同时表示一个int
};

int main()
{
	cout << sizeof(Test) << endl;
	return 0;
}
```

### 2.2 访问控制(Access control)

    我们不希望外部直接操作类内部的属性,因此我们为属性赋予访问权权限。

```
//class 和 struct 没有区别,除了若无说明,class默认private,struct默认public

class Student // self-definition type
{
	private:  //私有
	char *name;
	int id;
	public:   //公有
	void setAge(int aage);
	int getAge();
}
```

```
还有protected 在继承的时候再介绍
//对于外部为private,对于子类为protected
```

### 2.2* 想做一个软件,你需要学什么

硬件-->操作系统(Windows,ios,linux,Android,(进程、线程、API))-->应用知识(Database、socket、多线程、GUI)

1.数据库：sqllite  2.网络编程 udp socket  3. Linux多线程

### 2.3 构造和构析函数

```
class Student // self-definition type
{
	//constructor
	Student();
	//也可以自定义构造函数
	Student(int aid, char* aname)
	{
		this->id = aid;
		this->name = aname
	}
	//Overloading
	Student(int aid)
	{
		this->id = aid;
	}
	//试图模仿默认构造函数 -> 实际上起到了表明不需要传参的信息
	Student();
	//所以不是默认构造函数哦!!!!!
	private:  //私有
	char *name;
	int id;
	public:   //公有
	void setAge(int aage);
	int getAge();
}
int main()
{
	Student s(102, "zhangsan");
	Student s1(102);
	//构造函数的隐式调用
	//应当充分考虑对象的多样性,准备不同的构造函数应对不同的对象产生 ---> Overloading
}
//类不允许不初始化->存在默认构造函数,但是什么也不干
//运行区内存: 堆和栈(存放所有局部变量)
//局部变量不需要程序员来删除,会自行消亡 -> 特点: 快 缺陷: 变量生命周期不可控
//申请堆内存: malloc 释放:free (C语言) 特点: 1.人为控制生命周期(典型如链表) 2.内存长度不固定 3.比栈慢
//C++: 申请堆内存:new 释放:delete   Java: 变量全放堆区 & 自动释放
//链表解决了内存动态增长的问题: 数组定长,浪费内存或者会爆内存
//如果不释放,系统会自己杀内存(或者导致机器死机) -> 内存泄漏
```

```
class Test
{
	//attribute
	int i;//value
	int *p;//handle句柄
	public:
	Test(int ai,int aj);
	~Test();//desructor
}
Test::Test
{
	i = ai;
	p = new int(aj);
	//问题: 这个空间什么时候释放?
	//如果忘记释放: 内存泄漏(memory leak)->严重影响系统的性能
	//析构函数 destructor
}
//实际情况
class Wheel
{
	int i;
};
class SelfDrivingSys
{
	int ii;
};
class Car
{
	Wheel w[4];//value
	SelfDrivingSys *p_sys;//handle
	//使用变量还是句柄? 看是否为必有成员 -> 否则如果没有的话,就浪费空间了。
}
```

```
关于析构函数的一点小Tips
//1.析构会重载吗?
//答:不会,析构函数也不需要传参
//2.如果构造里有动态内存分配(new),那么一定有析构
//答:对。
//3.如果构造里面没有new,就不需要析构
//答:不对。析构会释放本类一切内存,包括中间产生的。
//4.不允许delete野指针,也不许重复delete指针
class Test
{
	//attribute
	int i;
	int *p;
	public:
	Test(int ai,int aj);
	Test(int ai);
	~Test();//desructor
}
Test::Test(int ai)
{
	i = ai;
	p = NULL;//必须要加上! 即使没值可以赋,也要赋值!决不允许出现野指针!!!
}
Test::Test(int ai,int aj)
{
	i = ai;
	p = new int(aj);
}
```

### 2.4 拷贝构造

```
class Test
{
	int i;
	int* p;
public:
	Test(int ai,int aj);
	Test(const Test& t);//参数是一个Test引用常量
}
Test::Test(int ai,int aj)
{
	i = ai;
	*p = aj;
}
Test::Test(const Test& t)
{
	this->i = t.i;
	this->j = new int(*t.j);//自己新建了内存空间,存放了t.j指向的值
}
//bitewise copy 默认就是 浅拷贝
//如果有指针,会产生一个尴尬的问题!
//同一块内存被两个handle指向!!
//问题的实质是: 你copy了一个指针!你让t1和t2互相影响了!
//而且会产生重复释放的问题
//logical copy 深拷贝
//实现原理:你自己写一个好用的拷贝,别用系统默认的

int main()
{
	Test t1(1,2);
	Test t2(t1);//拷贝构造(默认存在,不需要自己写)(和构造函数形式相似)
}
```

```
void fun(Test t)//pass by value(使用了拷贝构造)
{
	return;
}
void fun(Test& t)//pass by pointer
{};
//   pass by value        vs.      pass by address
//功能   input                       input/output✔ 
//性能   sizeof(value)               sizeof(int)✔
//其他   拷贝构造                      不需要拷贝构造✔
//conclusion : never pass by value
//针对self-definition type <=> build-in type(内嵌类型) 如int

int main()
{
	Test t1(1,2);
	fun(t1);//如何传值进函数?->拷贝构造! 此处也会调用拷贝构造
}
```

```
//一个常用的方法
class Test
{
	int i;
	int* p;
	Test(const Test& t);
	//将拷贝构造私有声明
	//强迫所有使用者不能使用该类的拷贝构造!
public:
	Test(int ai,int aj);

}
Test::Test(int ai,int aj)
{
	i = ai;
	*p = aj;
}
```

```
一、
int * fun2()
{
	int a = 10;
	return &a;
	//不要返回一个局部变量的地址!函数结束了，a就释放掉了。
}
二、
//fly pointer 一定要初始化！！没有就赋值为NULL
三、
//re-free 不要重复释放
标准写法：为了防止重复释放
	if（p!=NULL)
	{
		free(p);
        p = NULL;
	}
四、
int main()
{
	Test *p = new Test(1);
	int m=10;
	p->j = &m;
	//p->j 和 m 不是同一个生命周期！！
	//m为局部变量，而p->j是new出来的，不知道什么时候m就死了，但是p还活着，会引发大问题
}
```

### 2.5 静态(static)

```
1. static local var; 静态局部变量 保值,计数
void fun()
{
	static int i=0; //实现了类似全局变量的效果
	                //只被生成一次,程序结束释放
	i++;
	cout <<i<< endl;
}
//和全局变量的区别
//全局变量在程序开始前生成
//static变量在程序运行中生成
   
2.static fun(); 静态函数
//一旦将文件修饰上static,使得本函数'只在'本文件中能够被调用
//编外:不同文件全局变量
    (extern) void str2int();//函数的默认连接区域是全部区域
    extern int g_i;//变量的默认连接区是本文件,如果不是在本文件出现的全局变量,那么必须加extern


3.static global int;静态全局变量 //仅允许本文件使用
//感觉没必要!
以上为C语言就有的
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
4.static data member;
5.static function member
class Test
{
	static int i;//static修饰
	int j;
public:
	Test(int aj);
	static void fun();//可以不需要实例化,直接用类名访问!
};
void Test::Test(int aj)
{
	//i = ai; 不允许构造共享空间
	j = aj;
}
static void Test::fun()
{
	//一旦函数被static,函数内部只能访问静态成员,失去了this指针!
}
int Test::i = 100;
int main()
{
	Test t1(1,2);
	Test t2(2,3);
	//如果想使t1,t2互相感知,可以使用 全局变量->全局互相感知的终极方案
	//但是少用全局变量!!!会很容易错
	Test::fun();//类名、对象都直接调用
	return;
}
//可不可以使t1,t2共享一部分空间,以达到这个目的?
```

```
拓展: 这门课结课,可以接下来去学
design pattern 设计模式 23种
framework 框架 如QT
//单件模式 ----> 我们希望某一个类在全局只拥有唯一一个对象
class Single
{
	static Single* self;
	Single();
	public:
		static Single* get_instance();
};
Single* Single::get_instance()
{
 	if(self == NULL)
 	{
 		self = new Single();
 	}
 	return self;
}
Single* Single::self == NULL;
```

### 2.6 常量(const)

```
//const return value 

void fun(const int *i)
{
	//(*i)++; 常量不能赋值!不能做左值(left value);
}

class ClassA
{
public:
	void fun();
};
class ClassB
{
public:
	const A other();//const return value
	//像int double这种内嵌类型默认为常量
}
int main()
{
	ClassB b;
	b.other().fun();//常量不能作左值,b.other返回const A,不能连锁调用了
	return;
}
```

```
class Const
{
	//const int i;
	//enum{tcp,udp};//枚举,给类型取一个有意义的名字
public:
	void fun() const;//声明此函数可以被常量对象直接调用
}
void Const::fun() const
{
	//要求函数内部只能进行读操作,不能写入
	return;
}
int main()
{
	const Cosnt c;//创建了一个常量对象(不能修改)
	c.fun();//会报error,因为不敢让常量对象调用函数->必须声明此函数为常量函数->在类体内部写
	return 0;
}
```

```
class Test
{
	int  *p;
public:
	Test();
};
void Test::Test()
{
	p = new int;//先析构,把类申请的内存先释放
				//再free,将类本身内存释放
				//否则内存会泄露
}
int main()
{
	//malloc free 库函数 C
	//new delete operator(运算符) C++
	//Test* p1 = (Test*)malloc(sizeof(Test));//不会调用test的构造函数
	Test* p1 = new Test;
	//new = malloc(分配空间) + constructor(构造对象)
	//delete = destructor + free
	return 0;
}
```

## 作业一

自学:UDP socket,完成两个计算机(或两个程序)之间的内容收发
发送内容1 : 一个字符串
发送内容2 : 一个数据对象

## 3. 继承

inheritance & composition

注重reuse 代码的复用

### 软件的定价

```
//小插曲: 软件的定价
//1.人月(需要多少程序员工作多少个月)
//2.代码行数
//没有合理的计算方法,就随便吧
```

### 子类函数调用问题

```
//computer
对于复杂概念->分类(inheritance) & 组成(composition)
class Computer
{
	int price;
	char* brand;
public:
	int get_price();
	void service();
}
int Computer::get_price
{
	return this->price;
}
void Computer::service()
{
	cout<<"SERVICE"<<endl;
}
//inheritation
class Macbook: public Computer
{
	//个性与共性->继承了Computer的共性
}
void Macbook::service
{
	if(year<1) cout<<"new Macbook"<<endl;
	else Computer::service(); //复用了Computer的方法
	//若父类定义了方法A,子类重定义了A,但是子类一定会用到父类的A!
	//你既然选择了对方是你的父类,为什么不用父类的方法?
}
int main()
{
	Macbook mac;
	mac.get_price();
	return 0;
}
```

```
//继承要小心重载
class Base
{
public:
	void fun();
	void fun(int i);
};
void Base::fun(){}
void Base::fun(int i){}
class Drive:: pubic Base
{
	void fun();
	//子类没有定义父类的重载
	//要不然都改,要不然别改
};
int main()
{
	Drived d;
	d.fun();
	d.fun(1);//爆error 因为子类没有定义父类的重载,无法判断你想干嘛
}
```

### 继承中的构造构析问题

```
//类之间互相调用的构造构析
class Engine
{
public:
	Engine(int i);//挤掉了默认构造
	void run();
};
Engine::Engine(int i)
{
	cout<<"construct Engine"<<endl;
}
void Engine::run()
{
	cout<<"Engine is running"<<endl;
}
class Car
{
	Engine e;
	//Engine *p;
	//复习一下:当一个车必然有引擎的时候,就直接用对象,不然用指针,维持一个handle就可以了
public:
	void run();
}
//为什么要使用构造初始化列表？
//(组成的)构造初始化列表,此时e有名字
Car::Car() : e(1)
{
	cout<<"construct car"<<endl;
}
void Car::run()
{
	e.run();//reuse 复用成员类的代码,不会搞混car和engine的
	cout<<"car is running"<<endl;
}
```

```
//父类
class Base
{
	int i;
public:
	Base();
};
Base::Base(){
	cout<< "construct Base";
}
//子类
class Drived:: pubic Base
{
	int j;
public:
	Drived();
};
Drived::Drived(){
	cout<< "construct Drived";
}
int main()
{
	Drived d;
	Base b;
	cout<<sizeof(Drived)<<endl;//输出:8个字节,说明有i又有j->创建子类对象会不会调用父类构造函数呢?
							   //当然调用->顺序:先父类后子类 析构顺序:先子类后父类
}
```

```
👆改造版
//父类
class Base
{
	int i;
public:
	Base(int ai);//注意
};
Base::Base(){
	cout<< "construct Base";
}
//子类
class Drived:: pubic Base
{
	int j;
public:
	Drived(int ai, int aj);
};
//construction initialization list (继承的)构造初始化列表
Drived::Drived(int ai, int aj)::Base(ai)//指定父类构造
{
	j = aj;
	cout<< "construct Drived";
}
/*
Drived::Drived(int ai, int aj):Base(ai),j(aj)//可以在列表初始化成员变量
{											 //类内的常量必须放在构造初始化列表里面初始化
	cout<< "construct Drived";
}
*/
int main()
{
	Base b;
	Drived d;//报错,因为父类没有默认构造了,所以必须指明调用哪个父类构造对象
	cout<<sizeof(Drived)<<endl;
}
//总结:
//1. 子类对象构造时会先调用父类构造
//2. 如果构造子类对象时不希望用父类的默认构造,应用构造初始化列表表明如何调用构造函数
```

```
class Test
{
	int i;
	int j;
public:
	Test(int a);
};
Test::Test(int a):i(j),j(a)//i为垃圾,j为4
{}
Test::Test(int a):j(a),i(j)//i为垃圾,j为4
{}						   //初始化顺序不是按照你写的顺序决定的,而是按照定义的顺序初始化的
int main()
{
	Test t(4);
}
```

### 继承中的访问控制

```cpp
class Base
{
public:
	void fun();
	void fun(int i);
};
void Base::fun(){}
void Base::fun(int i){}
class Drive:: pubic Base//这里的public是什么意思?不加会怎么样
{						//默认为private,所以如果不加就会削弱子类对父类的访问权限,把父类public削弱为						  //private,也就是完全访问不了父类了,仅仅为了维持语法的正确性在继承
	void fun();
};
int main()
{
	Drived d;
	d.fun();
	return 0;
}
```

```
//访问控制 -> protected
对外界为private,对子类为public
```

### 多重继承

```cpp
class base1
{
public:
	void f();
};
class base2
{
	void h();
};
class Drived::Base1, Base2
{
};		//既可以用f,也可以用h
//那万一Base2里面也有f怎么办?->报错
//常见的糟糕继承:菱形继承
//在你成为一个高手之前,千万不要轻易多重继承
//解决方法
class Drived::Base1, Base2
{
	base2 b2; //一个类内的datamember一定是它的属性吗? 
public:		  //b2仅仅是为了代码重用,没有意义
	void h();
};
void Drived::h
{
	b2.h();
}
```

## 4. 多态

### Virtual 关键字实现多态

```C++

//class Pet
{
	int age;
	char *name;
public:
	/*virtual*/ void Speak(); //late banding 1
				  //借助虚函数实现多态
}
void Pet::Speak()
{
	cout<<"Pet::Speak"<<endl;
}
//猫
class Cat : public Pet
{
public:
	Speak();
}
void Cat::Speak()
{
	cout<<"miao"<<endl;
}
//狗
class Dog : public Pet
{
public:
	void Speak();//父类virtual了子类会自动进行，但是一般要写出来强调。
}
void Dog::Speak()
{
	cout<<"wang"<<endl;
}
//banding绑定：将函数的一次调用与函数入口地址相映射的过程，称为绑定。（由链接器负责）
//early banding 前绑定:
void Needle(Pet& pet)//late banding 2
{
	//传入dog,那么将执行dog的speak方法？ ---> NO!此处为前绑定，只会执行父类的方法
	//我们希望：
	/*
		if dog dog::Speak;
		if cat cat::Speak;
	*/
	//我们需要：late banding
	pet.Speak();//多态的体现
}
//主函数
int main()
{
	Dog dog;
	//Upcasting 向上类型转换：多态的前提 ->Downcasting is dangerous
	Needle(dog);//dog 继承自 Pet , 可以这么用吗？
		    //可以！保证了Needle的复用
		    //子类不能削弱父类的接口
	return 0;
}

```

```cpp
//虚指针的初始化问题
class Pet
{
	int age;
	char *name; 
public:
	virtual void Speak(); //借助虚函数实现多态
	//virtal void Sleep(); //无论写几个虚函数，都只占4字节，在对象的前四个字节
				//因为只会生成一个指针，指向虚函数表（如果是子类，那么子类会有一个自己的函数表）
				//证明：任何子类实例的前四个字节相同
				//函数名是指向函数运行位置的指针
}
//never pass by value！！！！！！
void Needle(Pet& pet)//如果去掉引用符号就错了！如果是传值，会调用Pet的拷贝构造函数,虚指针会被初始化为Pet的虚指针。
{			//虚指针是在构造函数内隐式初始化的
	pet.Speak();//多态的体现
}
int main()
{
	Dog dog1;
	Cat cat1;
	memcpy(dog1,cat1,4);//通过把dog对象的前四个字节替换为猫的，让狗miaomiao叫
			    //在内存里以字节为单位拷贝用memcpy
	Needle(dog1);//miaomiao
	dog1.Speak();//wangwang//此处没有多态，没有upcasting，就是前绑定
}
```

![1649769916284.png](image/C++与面向对象编程/1649769916284.png "图解")

### 纯虚函数

```cpp
//纯虚函数 -> 解决抽象类不便于初始化的问题
class Pet
{
	int age;
	char *name; 
public:
	virtual void Speak() = 0; //纯虚函数
}
//若一个类中存在纯虚函数，称这个类为抽象类abstract class
//抽象类不能实例化!子类如果不去实现纯虚函数,那么也将不能实例化.
//抽象类的子类必须实现所有方法 错,只不过不实现不能实例化
//两大作用:
//1. 约定一个类家族的共性行为 //Java的抽象类
//如果不实现抽象类的方法,那么将无法实例化
//2. 连接本不相关的类家族,抽象其共性行为 //Java的接口
class animal//动物类
{};
class Bird: public animal, public FlyObject
{
	void fly();
};
class machine//机器类
{};
class airplane:public machine, public FlyObject
{ 
	void fly();
};
//共性:都能飞
class FlyObject
{
	virtual void fly();
};
void scan(FlyObject& obj)//scan 想能够传入所有能飞的对象,怎么办? 纯虚函数+多重继承(辩证的理解)
{
	obj.fly();
}
//Java: 一个类只能有一个父类(杜绝多重继承),但是能实现多个接口(FlyObject只能算一个接口(interface))
//选择父类的原则:选择被upcasting最多的父类
```

```cpp
//考虑构造函数
class Base
{
public: 
	Base();
	virtual ~Base();
};
Base::~Base(){};
class Drived
{
public:
	Drived();
};
Drived::~Drived();
void fun(Base* p)
{
	delete p; //如果Base析构没虚, 这里只会调用父类的.
}
int main()
{
	Base b;
	Drived d;
	Base *p1 = new Base;
	Drived *p2 = new Drived;
	Base *p3 = new Derived;
	//new谁就调谁的构造函数,非常直觉,不是虚函数
	//但是析构函数需要是虚函数!
	fun(d);
	return 0;
}
//多态有损性能 对,但是带来了很多好处!
//若类内有一个虚函数,应该尽可能写虚函数 对.只有构造函数(?)和static不能写虚函数

```

## 5. 设计模式

## 6. Windows Programming

```
//异常处理Exception
void fun(int m) throw(int)
{
	if(m==5)
	   throw 1;
}
void fun(int* p)
{
	//头文件 #include<assert.h>
	assert(p!=NULL) //用于debug！避免出现程序卡死。
			//可以通过定义宏 #NDEBUG 使代码中所有断言失效。
	if(p == NULL)
	{
		cout<<"sorry"<<endl;
	}
}
int main()
{
	fun(5);
	catch(int e)
	{
		if(e==1)
			cout<<"sorry!"<<endl;
	}
	catch(...)//接受一切异常！
	{
		cout<<"sorry!"<<endl;
	}
}




```
