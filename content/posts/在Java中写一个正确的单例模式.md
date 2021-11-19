---
title: "在Java中写一个正确的单例模式"
date: 2021-11-19T19:00:36+08:00
draft: false
tags:
- Java
- 设计模式
---

> 本文参考极客时间每日一课[《在Java中如何写一个正确的单例模式？》](https://time.geekbang.org/dailylesson/detail/100044001)

## 1. 单例/单例设计模式



一个类只允许创建一个对象（实例），这个类就是一个单例类。这种设计模式叫做单例设计模式，简称单例模式。



实现一个单例类，需要关注：

1. 构造函数私有，避免通过new关键字创建实例
2. 创建时的线程安全问题
3. 是否支持延迟加载
4. getInstance是否高性能（锁）



## 2. 单例模式的写法



### 2.1. 饿汉式

```java
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();

    private Singleton() {
    }

    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```



```java
public class Singleton {
    private static Singleton instance;

    static {
        instance = new Singleton();
    }

    private Singleton() {}

    public static Singleton getInstance() {
        return instance;
    }
}
```



类加载时，静态实例就已经创建好并初始化好了。

实例创建过程是线程安全的。

不支持延迟加载。

### 2.2. 懒汉式

#### 1. 线程不安全

```java
public class Singleton {
    private static Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```



#### 2. 线程安全，并发度低

```java
public class Singleton {
    private static Singleton instance;

    private Singleton() {}

    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```



同一时刻，最多只有一个线程可以执行`getInstance`方法。



#### 3. 线程不安全

```java
public class Singleton {
    private static Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                instance = new Singleton();
            }
        }
        return instance;
    }
}
```



多个线程同时通过了`if`判断，依然会产生多个实例。



### 2.3. 双重检查

```java
public class Singleton {
    private static volatile Singleton instance;

    private Singleton() {
    }

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```



1. 没有第一层if，多个线程只能串行执行，效率低。
2. 没有第二层if，多个线程同时通过第一层if判断后，会产生多个实例。



### 2.4. 静态内部类

```java
public class Singleton {
    private Singleton() {}

    private static class SingletonInstance {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return SingletonInstance.INSTANCE;
    }
}
```



`Singleton` 被加载时，并不会创建`SingletonInstance` 实例对象。只有当调用`getInstance`方法时`SingletonInstance`才会被加载，这时才会创建singleton。



双重检查和静态内部类的写法都不能防止通过反序列化的方式生成多个实例。



### 2.5. 枚举

```java
public enum Singleton {
    INSTANCE;
    
    public void method() {}
}
```



枚举写法的优点：

1. 写法简单，不需要自己考虑懒加载、线程安全等问题。
2. 线程安全，枚举值会在类加载的时候被完全初始化。
3. 可以避免反序列化生成多个实例
