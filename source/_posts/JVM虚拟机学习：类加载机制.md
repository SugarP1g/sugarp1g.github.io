---
title: JVM虚拟机学习：类加载机制
date: 2023-03-26 21:46:16
tags:
  - JAVA
  - JDK
  - JVM
categories: JAVA
---

虚拟机类加载机制学习
<!-- more -->

# 类加载器

从Java虚拟机的角度来看，只存在两种不同的类加载器：

- 启动类加载器（Bootstrap ClassLoader），是虚拟机自身的一部分
- 其他所有的类加载器，独立存在于虚拟机外部

从 Java 程序来看，一般都会使用以下三种系统提供的类加载器来进行加载：

- 启动类加载器
- 扩展类加载器
- 应用程序类加载器

## 启动类加载器（Bootstrap Class Loader）

这个类加载器负责加载：
- 存放在 `<JAVA_HOME>/lib` 目录
- 被 `-Xbootclasspath` 参数所指定的路径中存放的、而且是 Java 虚拟机能够识别的类库

## 扩展类加载器（Extension Class Loader）

这个类加载器是在类 `sun.misc.Launcher$ExtClassLoader` 中以 Java 代码的形式实现的。 它负责加载：
- `<JAVA_HOME>/lib/ext` 目录
- 被 `java.ext.dirs` 系统变量所指定的路径中的所有的类库

## 应用程序类加载器（Application Class Loader）

这个类加载器由 `sun.misc.Launcher$AppClassLoader` 实现的。

由于应用程序类加载器是 ClassLoader 类中的 getSystemClassLoader() 方法的返回值。

它负责加载用户类路径（ClassPath）上所有的类库，开发者同样可以直接在代码中使用这个类加载器。

# 类和类加载器

对于任意一个类，都必须由加载它的类加载器和这个类本身一起共同确立其在Java虚拟机中的唯一性。

每一个类加载器，都拥有一个独立的类名称空间。

比较两个类是否相等必须满足：

1. 类对应同一个Class文件
2. 由同一个类加载器加载

> 注：这里所说的类相等，包括代表类的 Class 对象的 `equals()` 方法、`isAssignableFrom()` 方法、`isInstance()` 方法的返回结果，也包括使用 `instanceof` 关键字做对象所属关系判定等各种情况。

```java
package com.zch;

import java.io.IOException;
import java.io.InputStream;

public class ClassLoaderTest {

    public static void main(String[] args) throws Exception {
        ClassLoader cl = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    String fileName = name.substring(name.lastIndexOf(".") + 1) + ".class";
                    InputStream in = getClass().getResourceAsStream(fileName);
                    if (in == null) {
                        return super.loadClass(name);
                    }
                    byte[] bytes = new byte[in.available()];
                    in.read(bytes);
                    return defineClass(name, bytes, 0, bytes.length);
                } catch (IOException e) {
                    throw new ClassNotFoundException(name);
                }
            }
        };

        Object obj = cl.loadClass("com.zch.ClassLoaderTest").newInstance();

        System.out.println(obj.getClass());
        System.out.println(obj instanceof com.zch.ClassLoaderTest);
    }
}
```

执行结果如下：

```shell
class com.zch.ClassLoaderTest
false
```

执行结果中，第一行可以看到这个对象是类 `com.zch.ClassLoaderTest` 的实例化对象；第二行输出发现这个对象与类 `com.zch.ClassLoaderTest` 做所属类型检查的时候返回了 false。

因为 Java 虚拟机中同时存在两个 ClassLoaderTest 类，一个是由虚拟机的应用程序类加载器加载的，另一个是我们自定义的类加载器加载的。

# 双亲委派模型（Parents Delegation Model）

下图展示了各种类加载器之间的层次关系。除了顶层的启动类加载器外，其余的类加载器都应有自己的父类加载器。

> 注：这里加载器之间的父子关系一般不是以继承关系来实现的，而是通常使用组合关系来复用父加载器的代码。

```
             启动类加载器
        Bootstrap Class Loader
                  |
             扩展类加载器
        Extension Class Loader
                  |
            应用程序类加载器
       Application Class Loader
                  |
        --------------------
        |                  | 
   自定义类加载器       自定义类加载器
User Class Loader  User Class Loader
```

## 双亲委派模型的工作过程

如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到最顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求时，子加载器才会尝试自己去完成加载。

## 双亲委派模型的代码实现

双亲委派模型的代码实现如下：

1. 先检查请求加载的类型是否已经被加载过，若没有则调用父加载器的 `loadClass()` 方法；
2. 若父加载器为空则默认使用启动类加载器作为父加载器；
3. 假如父类加载器加载失败，抛出 `ClassNotFoundException` 异常的话，才调用自己的 `findClass()` 方法尝试进行加载。

```java
protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    // 首先检查请求的类是否已经被加载过
    Class c = findLoadedClass(name);
    if (c == null) {
        try {
            if (parent != null) {
                c = parent.loadClass(name, false);    
            } else {
                c = findBootstrapClassOrNull(name);
            }
        } catch (ClassNotFoundException e) {
            // 如果父类加载器抛出 ClassNotFoundException
            // 说明父类加载器无法完成加载请求
        }
        if (c == null) {
            // 在父类加载器无法加载时
            // 再调用本身的findClass方法来进行类加载
            c = findClass(name);
        }
    }
    if (resolve) {
        resolveClass(c);    
    }
    return c;
}
```