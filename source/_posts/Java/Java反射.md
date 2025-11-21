---
title: Java反射与XML&JSON
date: 2022-10-21 10:47:31
categories:
    - Java
tags:
    - 反射
    - XML
    - JSON
---
# 反射

## 类的加载概述和分类

#### 1.类加载的概述

当程序要使用某个类时，如果该类还未被加载到内存中，则系统会通过加载，链接，初始化三步来实现对这个类进行初始化。

加载

就是指class文件读入内存，并且为之创建一个Class对象。任何类被使用时系统都会建立一个class对象。

链接

验证 是否有正确的内部结构，并和其它类协调一致。

准备 负责为类的静态成员分配内村，并设置默认初始化值。

解析 酱类的二进制数据中的符号引用替换为直接引用

初始化

默认初始化、显式初始化、构造方法初始化等等。

#### 2.加载时机

创建类的实例（Person p = new Person()）

访问类的静态变量，或者为静态变量赋值（p.静态变量 或者 p.静态变量 = 0；）

调用类的静态方法

使用反射方式来强制创建某个类或接口对应的java.long.Class对象

初始化某个类的子类

直接使用java.exe命令来运行某个主类



#### 3.类加载器的概述

负责将.class文件加载到内存中，并为之生成对应的Class对象，虽然我们不需要关心类加载机制，但是了解这个机制我们就能更好地理解程序的运行。

#### 4.类加载器的分类

Bootstrap ClassLoader 根类加载器

Extension ClassLoader 扩展类加载器

System ClassLoader 系统类加载器

#### 5.类加载器的作用

Bootstrap ClassLoader 根类加载器

也被称为引导类加载器，负责Java核心类的加载

比如System，String等。在JDK中JRE的lib目录下rt.jar文件中

Extension ClassLoader扩展类加载器

负责JRE的扩展目录中jar的加载

在JDK中JRE的lib目录下ext目录

System ClassLoader 系统类加载器

负责在JVM启动时加载来自java命令的class文件，以及classpath环境变量所指定的jar包和类路径

## 反射概述

Java反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；

对于任意一个对象，都能够调用它的任意一个方法和属性；

这种动态获取的信息以及动态调用对象的方法的功能称为java语言的反射机制。

要想解剖一个类，就必须要获取到该类的字节码文件对象

而解剖使用的就是Class类中的方法，所以要先获取到每一个字节码文件对应的Class类型对象。

#### 三种方式

1.Object类的getClass()方法，判断两个对象是否时同一个字节码文件

2.静态属性class，锁对象

3.Class类中静态方法forName(),读取配置文件

![](Java反射/WEBRESOURCE59d8c4c388d4fa76527dcf6a69038ba1.png)



![](Java反射/WEBRESOURCE7903b6f0673f48b511e7f09f5aa7aab0.png)

#### 通过反射获取带参构造方法并使用

Constructor

Class类的newInstance()方法是使用该无参的构造函数创建对象，如果一个类没有无参的构造函数，就不能这样创建了，可以调用Class类的getConstructor（String.class,int.class）方法获取一个指定的构造函数然后再调用Constructor类的newInstance("张三"，20)方法创建对象

#### 通过反射获取成员变量并使用

Field

Class.getField(String)方法可以获取类中的指定字段(可见的)，如果时私有的可以用getDeclaedField("name")方法获取，通过set(obj，"李四")方法可以设置指定对象上该字段的值，如果是私有的需要先调用setAccessible(true)设置访问权限，用获取的指定的字段调用get(obj)可以获取指定对象中该字段的值

#### 通过反射获取方法并使用

Method

Class.getMethod(String，Class...)和Class.getDeclareMethod(String,Class...)方法可以获取类中的指定方法，调用invoke(object，object...)可以调用该法ing发，Class.getMethod(”eat“)invoke(obj)Class.getMethod("eat"，int.class)invoke(obj,10)
