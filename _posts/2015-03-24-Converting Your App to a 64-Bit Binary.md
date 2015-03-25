---
layout: post
title: Converting Your App to a 64-Bit Binary
category: iOS
tags: [iOS,Build,64-bit]
---

这里概括一下，创建同时支持32-bit和64-bit运行时环境app步骤

1. **安装Xcode 5.0.1**
2. **打开我们的工程**.Xcode提示你升级你的工程，升级过程中，会产生一些新的错误和警告。这些错误和警告在我们编译64-bit的app是至关重要的
3. **将工程的最低支持iOS版本升级至5.1.1**.如果我们支持的系统版本低于5.1，我们将不能构建64-bit的工程
4. **将build setting->Architectures改成“Standard Architectures (including 64-bit).”**
5. **升级应用支持64-bit运行时环境**虽然编译错误和警告可以帮助我们简化这项工作，但是不能帮助我们全部解决所以问题，我们可以i这份文档提到的注意点针对我们的代码进行适配
6. **在64-bit真机上测试**.不可否认iOS模拟器对在开发阶段很好用，但是有一些改变，必须在真机上才能测试和重现（例如 函数调用）
7. **利用Instruments来检测我们应用的内存性能**
8. **提交审核同时支持两种架构的app**
本章的剩余部分描述了当移植一个 Cocoa Touch app到64位运行时环境中经常出现的问题。参考这些部分来帮助我们更好的适配我们的代码

##Do Not Cast Pointers to Integers
事实上，我们很少需要将指针类型强制转换成整型.在使用指针的过程中，我们需要确保所有的变量都足够大到能表示一个地址
举个例子，下述代码，在32-bit中可以输出正确的结果，因为在32-bit中，int型和指针类型都是32bits，所有可以正常运算。而在64-bit运行时中，指针类型，已经远远的大于int型，在赋值过程中会被截断，丢失一部分数据。在这里，我们可以简单的增加指针的值来达到同样的效果
{% highlight objective-c%}
int *c = something passed in as an argument....
int *d = (int *)((int)c + 4); // Incorrect.
int *d = c + 1;               // Correct!
{% endhighlight %} 
如果我们必须将指针转换成int型，我们可以使用*uintptr_t*类型来避免数据被截断。*注意,像这样通过将指针转换成整型，然后加以数学计算，最后再转换成指针类型的做法直接违背基础类型转换的规则。这样，当试图访问一个错位的指针时，会引发不可预知的编译器错误或处理器错误

##Use Data Types Consistently
很多常见的编码级错误都是由于我们在编码过程中没有保持数据类型一致导致的。即使编译器会在我们使用不一致的数据类型时给我们警告，然而关注这些变化也会在我们代码中对我们有所帮助  
在调用函数时，我们必须保证接收者和函数的返回值数据类型一致。如果类型不一致，就有可能产生数据截断的问题
{% highlight objective-c%}
long PerformCalculation(void);

    int  x = PerformCalculation(); // incorrect
    long y = PerformCalculation(); // correct!
{% endhighlight %} 

同样的问题，也发生在形参和实参的类型上
{% highlight objective-c%}
int PerformAnotherCalculation(int input);
    long i = LONG_MAX;
    int x = PerformCalculation(i);
{% endhighlight %} 
函数返回值的类型一定要和声明的类型一致，否则也会出现截断问题（返回值被截断）
{% highlight objective-c%}
int ReturnMax()
{
    return LONG_MAX;
}
{% endhighlight %} 
所有的例子都是假定int和long类型是一样的，在标准ANSI C中并没有做出如此假定，而且这样在64-bit中是明显错误的。默认情况下，当我们升级了工程，
*-Wshorten-64-to-32*编译选项就已经自动为我们添加了，这样的话，编译器可以在很多数据截断的情况下给出警告。如果你没有升级你的工程，你就需要手动的开启这个选项。BTW，你也可以包含 *-Wconversion*选项，来发现更多潜在的问题

在Cocoa Touch应用中，查看下述整型，确保它们在你的代码中是使用正确的：

* long
* NSInteger
* CFIndex
* size_t (the result of calling the sizeof intrinsic operation)

在32-bit和64-bit运行时中，*fpos_t*和*off_t*类型都是64bits大小的，所以禁止将他们赋值给int

###Enumerations Are Also Typed
在LLVM编译器中，枚举也能定义不同的大小。确保将枚举类型赋值给合适的类型

###Common Type-Conversion Problems in Cocoa Touch
Cocoa Touch, notably Core Foundation and Foundation, add additional situations to look for, because they offer ways to serialize a C data type or to capture it inside an Objective-C object.  
**NSInteger大小增至64bit**NSInteger在整个Cocoa Touch中都有广泛的使用；它在32位运行时环境下占用32bit，而在64位运行时环境下则占用64bit。所以当我们在接受NSInteger 类型时，必须保证接收方也是NSInteger 类型  
我们应该避免NSInteger和int之间的转换，下面有一些特殊的例子：  
* 将NSInteger转换成NSNumber或者由NSNumber转化成NSInteger时
* 用NSCoder 对NSInteger做Encoding和decoding时。特别的，当我们在64位的设备上对NSInteger 进行编码而在32位的设备上对其进行解码时，如果结果的值超出了32位的整型，则会抛出异常。
* 使用framework内定义的NSInteger类型数据。特别值得注意的是NSNotFound。在64位运行时，它的价值比一个int类型的最大范围大，所以当他被截断时可能会引起程序错误。

**CGFloat大小增至64bit**和NSInteger一样CGFloat的大小也增至了64位。我们不能将CGFloat和double、float互相赋值。
{% highlight objective-c%}
// Incorrect.
CGFloat value = 200.0;
CFNumberCreate(kCFAllocatorDefault, kCFNumberFloatType, &value);
 
// Correct!
CGFloat value = 200.0;
CFNumberCreate(kCFAllocatorDefault, kCFNumberCGFloatType, &value);
{% endhighlight %} 

##Be Careful When Performing Integer Computations
虽然最常见的就是数据截断问题，但是我们也会碰到一些和整数计算有关的问题。这部分包括一些补充书名，我们可用于更新自己的代码

###Sign Extension Rules for C and C-derived Languages
C类的语言使用一组符号扩展规则用以确定当变量被赋值一个足够大宽度的值时，是否将最高位作为符号位处理。符号扩展规则如下：  
1. 






















