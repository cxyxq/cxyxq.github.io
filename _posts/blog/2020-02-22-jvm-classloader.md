---
layout: post
title: 类加载器介绍
categories: [jvm]
description: 类加载器的介绍、双亲委派机制
keywords: jvm
typora-root-url: ../../
---

### 类加载器

![](/images/jvm/jvm-classloader.png)

#### 虚拟机自带的类加载器

##### 启动类加载器 Bootstrap

采用c++语言实现(只限于HotSpot)。其他类型的虚拟机(J9,JRockit)都有1个代表Bootstrap ClassLoader的Java类存在, 但是关键方法的实现还是通过JNI调用的C语言库，同样的这个Bootstrap ClassLoader的实例也无法被用户获取到

##### 扩展类加载器 ExtClassLoader

sun.misc.Launcher$**ExtClassLoader**负责加载/lib/ext下的类

##### 应用程序类加载器 AppClassLoader

sun.misc.Launcher$**AppClassLoader**负责加载用户类路径的classpath下的类，因为这个类是`ClassLoader.getSystemClassLoader()`方法的返回值，所以一般也称为**系统类加载器**

#### 用户自定义类加载器

用户可以继承`java.lang.ClassLoader`实现自定义的类加载器加载方式



#### 程序示例

```java
public static void main(String[] args) {
        String s = new String();
        ClassLoader stringClassLoader = s.getClass().getClassLoader();
        //如果类的加载器是Bootstrap加载器，则返回null
        System.out.println("String的classLoader: " + stringClassLoader); 

        HelloWorld helloWorld = new HelloWorld();
        /**
         * 返回加载HelloWorld这个类的 类加载器
         */
        ClassLoader hwCL = helloWorld.getClass().getClassLoader();
        System.out.println("HelloWorld cl: " + hwCL); 
        System.out.println("HelloWorld cl.parent: " + hwCL.getParent());
        System.out.println("HelloWorld parent.parent: " + hwCL.getParent().getParent());
    }
```

> String的classLoader: **null**
> HelloWorld cl: sun.misc.Launcher$**AppClassLoader**@18b4aac2
> HelloWorld cl.parent: sun.misc.Launcher$**ExtClassLoader**@6f94fa3e
> HelloWorld cl.parent.parent: **null**

### 双亲委派机制

当1个类收到类加载请求时，

1. 他首先不会尝试自己加载，而是把加载请求委派父类(上一层级)去加载，
2. 每一层的类加载器都是如此，因此所有的类加载请求最终都会传递到顶层的Bootstrap启动类加载器。
3. 只有当父类加载器反馈自己无法加载时(在它的加载路径下没有找到需要加载的Class), 子类加载器才会尝试去加载。

比如要加载`java.lang.String`这个类，不管哪个类加载去加载这个类，最终都会委派给顶层的Bootstrap启动类加载器去加载。这样保证了使用不同的类加载器最终得到的都是同一个`String`对象。