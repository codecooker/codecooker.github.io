---
layout: post
title: Java ClassLoader
category: Java
tags: [Java]
---

众所周知Java中的类是由`ClassLoader`负责加载的，最近定位了一个关于类加载的问题，顺便回顾了下Java中类的加载机制，对Java中提供的`ClassLoader`进行一一说明。  

### Java中提供的ClassLoader

开始说明该问题前不妨先看看如下代码:  

```java
public static void main(String[] args) {
    // 用户自定义类
    // sun.misc.Launcher$AppClassLoader@14dad5dc
    System.out.println(Application.class.getClassLoader());

    // 系统基本类
    // null
    System.out.println(String.class.getClassLoader());

    // ext目录下的类
    // sun.misc.Launcher$ExtClassLoader@1c20c684
    System.out.println(DNSNameService.class.getClassLoader());
}
```

由上述代码可以看出，用户自定义的类是有`AppClassLoader`加载，ext目录的类是有`ExtClassLoader`加载，系统基本类的加载器为`null`。查看Java手册得知，Java语言自带有3个加载器，分别为:  

* **Bootstrap ClassLoader**,最顶层的加载类，主要加载核心类库，%JRE_HOME%\lib下的rt.jar、resources.jar、charsets.jar和class等。另外需要注意的是可以通过启动jvm时指定-Xbootclasspath和路径来改变Bootstrap ClassLoader的加载目录。比如java -Xbootclasspath/a:path被指定的文件追加到默认的bootstrap路径中。
* **Extention ClassLoader**, 扩展的类加载器，加载目录%JRE_HOME%\lib\ext目录下的jar包和class文件。还可以加载-D java.ext.dirs选项指定的目录。
* **Appclass Loader**也称为SystemAppClass, 加载当前应用的classpath的所有类。


### 双亲委托

#### 简单模型

要弄明白各个`ClassLoader`是如何工作的，在这里要引入一个新的概念**双亲委托**,介绍**双亲委托**之前，我们先看*ExtClassLoader*、*AppClassLoader*的源码。从源码中，可以看出一个前提，每个`ClassLoader`内都包含一个`Parent ClassLoader`引用，可用通过`ClassLoader`中的`getParent()`获取。(用法后边在展开)  
`AppClassLoader`在加载类时会调用如下方法:  

```java
 public Class<?> loadClass(String var1, boolean var2) throws ClassNotFoundException {
    int var3 = var1.lastIndexOf(46);
    if (var3 != -1) {
        SecurityManager var4 = System.getSecurityManager();
        if (var4 != null) {
            var4.checkPackageAccess(var1.substring(0, var3));
        }
    }
    if (this.ucp.knownToNotExist(var1)) {
        Class var5 = this.findLoadedClass(var1);
        if (var5 != null) {
            if (var2) {
                this.resolveClass(var5);
            }
            return var5;
        } else {
            throw new ClassNotFoundException(var1);
        }
    } else {
        return super.loadClass(var1, var2);
    }
}
```
从源码中可以看出，首先`AppClassLoader`会判断class是否已经加载，如果已经加载过则从加载过的cache中直接返回，如果不存在则调用`super.loadClass`进行加载(需要注意的是，这里的`super.loadClass`只是调动父类方法，而不是`Parent`去加载)。  
跟着代码查看`super.loadClass`方法：  

```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }
            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);
                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```
可以很清楚的看到，`super.loadClass`同样在cache中查找class是否已经加载，如果加载了直接返回，如果没有加载的话，则调用`Parent`的`loadclass`方法加载，当`Parent`的`loadclass`为`null`时，再调用自身的`findClass`进行查找class，这样就形成了一个双向查找链，大致如下:  
![整体框架]({{ site.res }}/images/Java_ClassLoader/1.png)

1. 首先在cache中找，cache中找不到也不自己加载，而是委托Parent加载
2. Parent加载不到再委托子加载器按照反方向逐级加载  

如上，就是**双亲委托**的简单模型。

#### 谁是谁的Parent

同样，先看如下代码,Java虚拟机启动时的流程:   
```java
public Launcher() {
    Launcher.ExtClassLoader var1;
    try {
        var1 = Launcher.ExtClassLoader.getExtClassLoader();
    } catch (IOException var10) {
        throw new InternalError("Could not create extension class loader", var10);
    }

    try {
        this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);
    } catch (IOException var9) {
        throw new InternalError("Could not create application class loader", var9);
    }
    ...
}
```
在Java虚拟机启动的时分别加载了*ExtClassLoader*,*AppClassLoader*。    
分别打印各个loader的Parent，如下:  
```java
public static void main(String[] args) {
    // 用户自定义类,sun.misc.Launcher$AppClassLoader@14dad5dc
    System.out.println(Application.class.getClassLoader());
    // AppClassLoader->Parent,sun.misc.Launcher$ExtClassLoader@681a9515
    System.out.println(Application.class.getClassLoader().getParent());
    // ExtClassLoader->Parent,null
    System.out.println(Application.class.getClassLoader().getParent().getParent());
}
```
由结果可以看出，`ExtClassLoader`的`Parent`为null，并不代表`ExtClassLoader`没有`Parent`，而是因为他是`Bootstrap ClassLoader`,由于`Bootstrap ClassLoader`是启动加载器，属于Jvm的一部分，有C/C++编写，在Jvm中处理，所以Java中自然就为null，细心的同学可以在`Launcher.class`中发现线索，在`Launcher.class`中有如下配置：  
```java
private static String bootClassPath = System.getProperty("sun.boot.class.path");
```
Bootstrap加载器在启动时通过*sun.boot.class.path*,来加载启动路径下的class.

#### 为什么双亲委托

谈这个问题，不妨回过头来看看Java的class加载顺序有什么限制，系统类、扩展类都是有`Bootstrap ClassLoader`，`ExtClassLoader`加载，无论是否自定义自己的`ClassLoader`都无法实现覆写诸如`String`这些基本类的操作，即保留了灵活性，也保证了JVM的运行安全。

### 自定义ClassLoader

自定义ClassLoader可以有效的保证类隔离，避免互相冲突加载，影响业务功能，同时可以根据业务自身需求提供多种加载方式，例如从网络上加载对应的class，或者在加载class做一些预处理。  
定义自己的ClassLoader，同时需要指定Parent，完成**双亲代理**的双向链条，通常情况下我们可以指定为`AppClassLoader`,如果不指定则默认为`AppClassLoader`，如下。

```java
@Override
protected Class<?> findClass(String name) throws ClassNotFoundException {
    String fileName = getFileName(name);
    File file = new File(fileName);
    try {
        FileInputStream is = new FileInputStream(file);
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        int len = 0;
        try {
            while ((len = is.read()) != -1) {
                bos.write(len);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        byte[] data = bos.toByteArray();
        is.close();
        bos.close();
        return defineClass(name,data,0,data.length);
    } catch (IOException e) {
        e.printStackTrace();
    }
    return super.findClass(name);
}
// 获取要加载 的class文件名
private String getFileName(String name) {
    int index = name.lastIndexOf('.');
    if(index == -1){
        return name+".class";
    }else{
        return name.substring(index+1)+".class";
    }
}
```
其中最重要的一点就是调用`defineClass`方法，完成二进制文件到class的定义。  
至此，关于Java的ClassLoader总结到一段落。