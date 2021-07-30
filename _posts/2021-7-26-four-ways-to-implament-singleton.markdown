---
layout: post
title:  "单例模式，关键字级别详解"
date:   2021-7-26 08:00:00 +0800
category: Java
author: Java课代表
excerpt: 单例模式看似简单，实则包含了很多java基础知识，本文介绍了四种常规单例实现方式并从关键字级别做出了说明
---

## 0.前言

如果你去问一个写过几年代码的程序员用过哪些设计模式，我打赌，90%以上的回答里面会带【单例模式】。甚至有的面试官会直接问：说一下你用过哪些设计模式，单例就不用说了。你看，连面试官都听烦了，火爆程度可见一斑。

不过，看似简单的单例模式，里面蕴含了很多`Java`基础，日常开发过程中课代表见过很多不规范的，甚至是有问题的单例实现。所以整理此文，总结一下单例模式的最佳实践。

## 1、懒加载（懒汉）

所谓懒加载，就是直到第一次被调用时才加载。其实现时需要考虑并发问题和指令重排，代码如下：

```java
public class Singleton {

    private volatile static Singleton instance; //①

    private Singleton() { //②
    }

    public static Singleton getInstance() {
        if (instance == null) {//③
            synchronized (Singleton.class) {
                if (instance == null) {//④
                    instance = new Singleton();//⑤
                }
            }
        }
        return instance;
    }
}
```

这段代码精简至极，没有一个字符是多余的，下面逐行解读一下：

首先，注意到①处的`volatile`关键字，它具备两项特性：

> 一是保证此变量对于所有线程的可见性。即当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的。
>
> 二是禁止指令重排序优化。

这里解释一下指令重排序优化：

代码 ⑤ 处的`instance = new Singleton();`并不是原子的，大体可分为如下 3 步：

1. 分配内存
2. 调用构造函数初始化成实例
3. 让`instance`指向分配的内存空间

JVM 允许在保证结果正确的前提下进行指令重排序优化。即如上 3 步可能的顺序为1->2->3 或 1->3->2 。如果顺序是 1->3->2 ，当 3 执行完，2 还未执行时，另一个线程执行到代码 ③ 处，发现`instance`不为`null`，直接返回还未初始化好的`instance`并使用，就会报错。

所以使用`volatile`，就是为了保证线程间的可见性和防止指令重排。

其次，代码②处将构造函数声明为`private`目的在于阻止使用`new Singleton()`这样的代码生成新实例。

最后，当客户端调用`Singleton.getInstance()`时，先检查是否已经实例化(代码③)，未实例化时同步代码块，然后再次检查是否已实例化(代码④)，然后才执行代码⑤。两次检查的意义在于，防止`synchronized`同步过程中其他线程进行了实例化。

这就是著名的双重检查锁(Double check lock)实现单例，也即懒加载。

> TIPS: 
>
> 网上也有直接对`getInstance()`方法加锁的版本，这样大范围的方法级别加锁会导致并发变低，实际上第一次调用生成实例之后，后续获取实例根本不需要并发控制了。而本例的双重检查锁版本可以避免此并发问题。

## 2、预加载（饿汉）

与懒加载相对应，预加载是在类加载时就已经初始化好了，所以是天然线程安全的，代码如下：

```java
public class Singleton {

    private static final Singleton instance = new Singleton();// ①
    
    private Singleton(){}
    
    public static Singleton getInstance(){
        return instance;
    }
}
```

注意到 ① 处的类变量使用了`final`。

这里用`final`更多的意义在于提供语法约束。毕竟你是单例，就只有这一个实例，不可能再指向另一个。`instance`有了`final`的约束，后面再有人不小心编写了修改其指向的代码就会报语法错误。

这就好比`@Override`注解，你能保证写对方法名和参数，那不写注解也没问题，但是有了注解的约束，编译器就会帮你检查，还能防止别人乱改。

## 3、静态内部类

此方法和预加载原理相同，都是利用JVM类加载的特性实现天然的线程安全，不同之处在于，静态内部类做到了延迟加载。

```
public class Singleton {
    
    private static class SingletonHolder {
        private static Singleton instance = new Singleton();
    }
    
    private Singleton(){}

    public static Singleton getInstance() {
        return SingletonHolder.instance;
    }
}
```

`SingletonHolder` 是静态内部类，当外部类`Singleton`被加载的时候并不会创建任何实例，只有当`Singleton.getInstance()`被调用的时候，才会创建`Singleton`实例，这一切由 JVM 天然完成，所以既保证了线程安全，又实现了延迟加载。

## 4、枚举

没错，枚举可以实现单例，而且这种方式是《Effective Java中文版》第二版 中的推荐实现方式。代码极其简单：

```java
public enum Singleton {
    /**
     * 单例实例
     */
    INSTANCE;

    public void doSomeThing(){
        System.out.println("done");
    }
}
```

使用时直接`Singleton.INSTANCE.doSomeThing();`即可。

这里主要利用了枚举的如下两个特性：

* 枚举的构造器总是私有的，所以不必像前几种方式一样显式定义私有构造方法
* 枚举类中的每个值，都是实例（只有`INSTANCE`这一个实例）

除此之外，枚举还附带了一些额外好处：无偿地提供了序列化机制，还可以防止通过多次反序列化生成多个实例。

鉴于此，单例的最佳实践就是用枚举来实现。

## 5、总结

事实上，单例的写法并不止于本文所提的这 4 种，你可能还会看到很多其他变种，它们或多或少都存在一些缺陷，比如，懒加载方式将`synchronized`作用于整个方法上也能实现，但频繁加锁，释放锁会产生性能瓶颈，而完全去掉锁又会带来并发问题。

所以，只要吃透了文中列出的这 4 种单例方式，就能做到举一反三，见到别人写的单例也能一眼看出对错。

文中所列的 4 种单例模式，除了枚举之外，全都用到了`static`关键字，《Java 虚拟机规范》 规定，有几种情况必须立即对类进行“初始化”，其中涉及`static`的场景如下：

> 读取或设置一个类型的静态字段（被 final 修饰、已在编译期把结果放入常量池的静态字段除外）的时候。
>
> 调用一个类型的静态方法的时候。

懒加载，预加载和静态内部类正是利用了这两点特性。

对`static`关键字遗忘的同学可以参看我的另一篇文章：[《一题搞定static关键字》](https://javahelper.top/java/2020/07/01/learn-static-by-problem.html)

最后，再次强调一下，如果大家开发中需要手写单例，建议听从 Joshua Bloch在《Effective Java中文版》第二版 中的建议：

> 单元素的枚举类型已经成为实现 Singleton 的最佳方法



## 参考资料：

1、《Effective Java中文版》 Joshua Bloch 第二版 P15

2、《深入理解 Java 虚拟机》 周志明 第3版，P444-P448，P264

3、[深入浅出单实例SINGLETON设计模式](https://coolshell.cn/articles/265.html)

