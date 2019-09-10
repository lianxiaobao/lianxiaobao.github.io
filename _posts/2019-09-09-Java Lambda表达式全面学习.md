---
layout:     post
title:      java 8 特性
subtitle:   Java Lambda表达式全面学习，再不学会被淘汰
date:       2019-09-10
author:     BY xiaobao(微信：Bao1697047283)
header-img: img/post-bg-mi1.jpg
catalog: true
tags:
    - blog
    - 数据开发
    - java
    
---

# Java Lambda 表达式
本篇我们首先感受一下使用Lambda表达式带来的便利之处。
>Java Lambda表达式的一个重要用法是简化某些匿名内部类（`Anonymous Classes`）的写法。
>

## Lambda and Anonymous Classes(I)

本节将介绍如何使用Lambda表达式简化匿名内部类的书写，但Lambda表达式并不能取代所有的匿名内部类，只能用来取代函数接口（`Functional Interface`）的简写。先别在乎细节，看几个例子再说。

>例子1：无参函数的简写

如果需要新建一个线程，一种常见的写法是这样：

```java
// JDK7 匿名内部类写法
new Thread(new Runnable() { // 接口名
    @Override
    public void run() { // 方法名
        System.out.println("Thread run()");
    }
});

```

上述代码给`Tread`类传递了一个匿名的`Runnable`对象，重载`Runnable`接口的`run()`方法来实现相应逻辑。这是JDK7以及之前的常见写法。匿名内部类省去了为类起名字的烦恼，但还是不够简化，在Java 8中可以简化为如下形式：

```java
// JDK8 Lambda表达式写法
new Thread(
        ()-> System.out.println("Thread run()")// 省略接口名和方法名
);
```
上述代码跟匿名内部类的作用是一样的，但比匿名内部类更进一步。这里连`接口名和函数名都一同省掉`了，写起来更加神清气爽。如果函数体有`多行，可以用大括号括起来`，就像这样：

```java
// JDK8 Lambda表达式代码块写法
new Thread(
        ()-> {
            System.out.println("Thread-1 ");// 省略接口名和方法名
            System.out.println("Thread-2 ");
        }
);
```
>例子2：带参函数的简写

如果要给一个字符串列表通过自定义比较器，按照字符串长度进行排序，Java 7的书写形式如下：


