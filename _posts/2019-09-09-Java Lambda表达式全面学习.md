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

```java
// JDK7 匿名内部类写法
//通过内部类重载了Comparator接口的compare()方法，实现比较逻辑。
List<String> list = Arrays.asList("i","love","guo","jin");
Collections.sort(list, new Comparator<String>() {//接口名
    @Override
    public int compare(String o1, String o2) {//方法名
        if(o1 == null){
            return -1;
        }
        if(o2 == null){
            return -1;
        }
        return o1.length()-o2.length();
    }
});
```

采用Lambda表达式可简写如下：

```java
// JDK8 Lambda表达式写法
List<String> list = Arrays.asList("i","love","guo","jin");
Collections.sort(list,(o1, o2)->{
    if(o1 == null){
        return -1;
    }
    if(o2 == null){
        return -1;
    }
    return o1.length()-o2.length();
});
```
上述代码跟匿名内部类的作用是一样的。除了省略了接口名和方法名，代码中把参数表的类型也省略了。这得益于`javac的类型推断机制`，编译器能够根据上下文信息推断出参数的类型，当然也有推断失败的时候，这时就需要手动指明参数类型了。注意，Java是强类型语言，每个变量和对象都必需有明确的类型。

### 简写的依据
`能够使用Lambda的依据是必须有相应的函数接口`（函数接口，是指内部只有一个抽象方法的接口）。这一点跟Java是强类型语言吻合，也就是说你并不能在代码的任何地方任性的写Lambda表达式。实际上Lambda的类型就是对应函数接口的类型。`Lambda表达式另一个依据是类型推断机制`，在上下文信息足够的情况下，编译器可以推断出参数表的类型，而不需要显式指名。Lambda表达更多合法的书写形式如下：

```java
// Lambda表达式的书写形式
Runnable run = () -> System.out.println("Hello World");// 1
ActionListener listener = event -> System.out.println("button clicked");// 2
Runnable multiLine = () -> {// 3 代码块
    System.out.print("Hello");
    System.out.println("xiao bao");
};
BinaryOperator<Long> add = (Long x, Long y) -> x + y;// 4
BinaryOperator<Long> addImplicit = (x, y) -> x + y;// 5 类型推断

```
>1. 1展示了无参函数的简写；
>2. 2处展示了有参函数的简写，以及类型推断机制；
>3. 3是代码块的写法；
>4. 4和5再次展示了类型推断机制。
>
>

### 自定义函数接口
自定义函数接口很容易，只需要编写一个只有一个抽象方法的接口即可。


```java
// 自定义函数接口
@FunctionalInterface
interface consumerInterface<T>{
    void accept(T t);
}
```