---
layout: post
title: iOS特性修饰符(strong、weak...)
category: objective-c
tags: [objective-c,iOS]
---


iOS特性在修饰的时候我们经常会和下面几个修饰符打交道
{% highlight objective-c%}
strong、retain、assign、unsafe_unretained、weak、nonatomic、atomic、copy
{% endhighlight %} 
其中与内存有关的有strong、retain、assign、unsafe_unretained、weak，在使用过程中稍有不慎就会陷入内存泄露，搞清楚他们的作用很有必要  
在描述之前我们需要补充说明一些概念
{% highlight objective-c%}
//修饰变量我们用__strong形式的修饰符
__strong id a = nil;
__unsafe_unretained id b = nil;
__weak              id c = nil;
//修饰特性我们用不带下划线的，表示一样
@property (nonatomic,strong) MyObject* object1;
@property (nonatomic,assign) MyObject* object1;
@property (nonatomic,unsafe_unretained) MyObject* object1;
@property (nonatomic,weak) MyObject* object1;
{% endhighlight %} 

#### retain 和 strong
了解retain和strong的渊源，必须要先介绍下ARC（自动引用计数）。在xcode4.2之前开发者在开发代码的过程中和写C++代码一样，需要手动管理内存的分配和释放（MRC），从xcode4.2之后，引入了ARC，对开发者来说却是一个福音
>ARC,即自动引入计数，编译器会在程序编译前为我们对相应的对象的引用计数+1或者-1.当对象的引用计数为0的时候，触发回收机制。极大的降低了iOS开发的成本，由于是编译级别的优化，在效率方面也会比我们手动管理高出很多。并且可以避免我们手动管理时的遗漏。
retain是MRC时代的产物，与release对应。当给对象发送retain消息时，该对象的引用计数+1，release则-1.如果不retain的话，我们可以理解为是一个弱引用  


可以看下面代码  
{% highlight objective-c%}
id obejct = [[[NSObject alloc]init]autorelease]; //初始化会调用retain，此时引用计数为1
[obejct retain];                                 //引用计数器+1
[obejct release];                                //对object的引用计数-1
NSLog(@"object retainCount = %lu" , (unsigned long)[obejct retainCount]);
{% endhighlight %} 
在ARC中，对对象的引用我们用strong修饰，为强引用，引用计数器+1，编译器会帮我们加上retain和release，所以不需要自行添加。我们可以这样理解，ARC中得strong和MRC中得retain得作用是一样的，只是在ARC中，编译器会自动为我们加上retain和release  
<kp>注意：retain和strong是通用的，效果一样</kp>  

#### weak 和 unsafe_unretained
weak和unsafe_unretained都是ARC中常见的修饰符，两者都表示弱引用。区别在于当对象释放后这两个修饰符修饰的指针的值的变化  
{% highlight objective-c%}
id object0 = [NSObject new];
__weak id object1 = object0;
object0 = nil;
NSLog(@"%@",object1);
{% endhighlight %} 
输出结果为:  
<kp>(null)</kp>
{% highlight objective-c%}
id object0 = [NSObject new];
__unsafe_unretained id object1 = object0;
object0 = nil;
NSLog(@"%@",object1);
{% endhighlight %} 
输出结果为：  
<kp><NSObject: 0x7f8c2a47ca20></kp>
看似没有我们的指针还指向对象，我们来尝试对该对象发送一个消息  
{% highlight objective-c%}
MyObject *object0 = [MyObject new];
__unsafe_unretained MyObject* object1 = object0;
object0 = nil;
[object1 respondsToSelector:@selector(sayHello)];
NSLog(@"%@",object1);
{% endhighlight %} 
对，没错，crash了，所以我们可以知道weak和unsafe_unretained修饰的指针都不会改变对象的引用计数，区别在于当对象释放后weak修饰的指针会被设置为nil，而unsafe_unretained则不会 

#### assign
assign修饰简单的赋值，例如基础变量int、float等。但当我们用assign修饰对象时会发生什么情况呢？
{% highlight objective-c%}
@property (atomic,assign) MyObject* object1;

MyObject *object0 = [MyObject new];
self.object1 = object0;
object0 = nil;
[self.object1 sayHello];
NSLog(@"%@",self.object1);
{% endhighlight %} 
对象对释放，但是指针变量会被输出。

这里我们会有几个疑问，为什么对象被释放了，但是程序有时候竟然能执行成功？  
因为当我们的对象被销毁后，其只是对该块内存标志为可用，但是我们的指针变量还指向的该块内存。如果该块内存还没有被来得及重复使用，则我们的调用即是能成功的。但是我可以肯定这样做是很不安全的  

#### copy
在赋值时使用传入值的一份拷贝,需要实现NSCopying协议

#### nonatomic和atomic
atomic是Objc使用的一种线程保护技术，基本上来讲，是防止在写未完成的时候被另外一个线程读取，造成数据错误。而这种机制是耗费系统资源的，所以在iPhone这种小型设备上，如果没有使用多线程间的通讯编程，那么nonatomic是一个非常好的选择。指出访问器不是原子操作，而默认地，访问器是原子操作。这也就是说，在多线程环境下，解析的访问器提供一个对属性的安全访问，从获取器得到的返回值或者通过设置器设置的值可以一次完成，即便是别的线程也正在对其进行访问。如果你不指定 nonatomic ，在自己管理内存的环境中，解析的访问器保留并自动释放返回的值，如果指定了 nonatomic ，那么访问器只是简单地返回这个值。

#### 注意事项
应该特别注意的是，当特性用retain或者strong修饰时，每次调用特性，该特性对应的对象的引用计数器都会被+1，如果是在MRC下，需要手动release。在ARC下不用则不需要处理。


