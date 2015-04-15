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
**NSInteger大小增至64bit**  
NSInteger在整个Cocoa Touch中都有广泛的使用；它在32位运行时环境下占用32bit，而在64位运行时环境下则占用64bit。所以当我们在接受NSInteger 类型时，必须保证接收方也是NSInteger 类型  
我们应该避免NSInteger和int之间的转换，下面有一些特殊的例子：  
* 将NSInteger转换成NSNumber或者由NSNumber转化成NSInteger时  
* 用NSCoder 对NSInteger做Encoding和decoding时。特别的，当我们在64位的设备上对NSInteger   进行编码而在32位的设备上对其进行解码时，如果结果的值超出了32位的整型，则会抛出异常   
* 使用framework内定义的NSInteger类型数据。特别值得注意的是NSNotFound。在64位运行时，它的价值比一个int类型的最大范围大，所以当他被截断时可能会引起程序错误  

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
1. 将无符号值提升至一个大的数据类型时做无符号扩展（Unsigned values are zero extended (not sign extended) when promoted to a larger type.）  
2. 有符号值始终做符号扩展，即使结果是一个无符号值(Signed values are always sign extended when promoted to a larger type, even if the resulting type is unsigned.)  
3. 常量将会被当做能处理它的最小数据类型处理，16进制的数字可能被当做*int*、*long*、*long long*、*signed*、*unsigned*处理，而十进制则始终被当做有符号的*int* (Constants (unless modified by a suffix, such as 0x8L) are treated as the smallest size that will hold the value. Numbers written in hexadecimal may be treated by the compiler as int, long, and long long types and may be either signed or unsigned types. Decimal numbers are always treated as signed types)  
4. 大小相同的有符号值和无符号值求和，结果为无符号值(The sum of a signed value and an unsigned value of the same size is an unsigned value)  

{% highlight objective-c%}
int a=-2;
unsigned int b=1;
long c = a + b;
long long d=c; // to get a consistent size for printing.
 
printf("%lld\n", d);
{% endhighlight %} 
*问题：*上述代码在32-bit运行时环境运行时，结果为*-1 (0xffffffff)*，而在64-bit运行时环境得出的结果为*4294967295 (0x00000000ffffffff)*，这样就有可能不是我们所期望的结果  
*原因:*有符号数值和无符号数值求和结果是一个无符号值（上述第4个规则）
*解决方案：*将b转换成long型，这个强制转换将b在计算前强制转换成64bit，这样的话结果就为-1

{% highlight objective-c%}
unsigned short a=1;
unsigned long b = (a << 31);
unsigned long long c=b;
 
printf("%llx\n", c);
{% endhighlight %} 
*问题：*我们期望的结果是0x80000000（在32-bit中运行），然而在64-bit中却得到了0xffffffff80000000
*原因：*这部分文档本人也看了一会儿，我是这么理解的，如果有不对之处还望海涵。当我们的常量1即是一个signed int类型，当我们将这个常量赋值给unsigned short 时其结果还是一个unsigned类型。这时我们将一个unsigned 类型做左移操作然后赋值给 unsigned long 会做符号扩展。所以我们得到的结果就是0xffffffff80000000


###Working with Bits and Bitmasks


##Create Data Structures with Fixed Size and Alignment
当我们在32-bit和64-bit之间共享数据时，我们需要构造在32-bit和64-bit中表现形式相同的数据。大多数情况下，我们将数据存储至文件或是通过网络传递数据到一个设备时，会面对不同的运行环境，而且也应该注意到用户可能会从一个32-bit的设备读取数据然后存储至一个64-bit的设备。所以构造表现形式相同的数据是我们必须解决的问题  

###使用准确的整型数据类型
![ C99 explicit integer types]({{ site.url }}/images/Converting_Your_App_to_a_b4-Bit_Binary/Snip20150415_1.png)

###注意64-bit整型对齐
在64-bit运行时中，所有的64-bit整数类型的对齐方式从4字节到变为了8个字节。即使您明确指定每个整数类型，这两种结构可能仍然无法在两个运行时保持相同。
{% highlight objective-c%}
struct bar {
    int32_t foo0;
    int32_t foo1;
    int32_t foo2;
    int64_t bar;
};
{% endhighlight %} 

当我们在32-bit的编译器编译上面的代码时，变量bar是从这个结构体的第12个字节开始分配空间的，而同样的代码我们用64-bit的编译器进行编译时，变量bar则是有第16个字节开始，为了保证bar是8个字节对齐的，编译器会在变量foo2后边加上4个字节的填充

我们可以在定义数据结构时，将占用内存大的放在前面而将占用内存小的放在后面，这样布局的话可以减少过多的数据填充，如下：
{% highlight objective-c%}
struct bar {
    int64_t bar;
    int32_t foo0;
    int32_t foo1;
    int32_t foo2;
};
{% endhighlight %} 

当然我们也可以使用下述方式，来强制干预对齐方式
{% highlight objective-c%}
#pragma pack(4)
struct bar {
    int32_t foo0;
    int32_t foo1;
    int32_t foo2;
    int64_t bar;
};
#pragma options align=reset
{% endhighlight %} 


##分配内存时使用sizeof
由于在不同的运行时环境，变量所占用的内存大小有所差别。为了保证我们程序的通用性，我们必须使用sizeof关键字来分配内存，而不是采用硬编码的形式来决定分配内存的大小


##更新字符串的格式化输出
不同类型整形数据的格式化输出格式，参见下表  
1. 标志格式
![ Standard format ]({{ site.url }}/images/Converting_Your_App_to_a_b4-Bit_Binary/Snip20150415_2.png)
2. 额外的格式
![ Additional]({{ site.url }}/images/Converting_Your_App_to_a_b4-Bit_Binary/Snip20150415_3.png)

我们举个例子，当我们要格式化输出一个*intptr_t*类型的变量和一个指针时，可参见如下代码
{% highlight objective-c%}
#include <inttypes.h>
void *foo;
intptr_t k = (intptr_t) foo;
void *ptr = &k;
 
printf("The value of k is %" PRIdPTR "\n", k);
printf("The value of ptr is %p\n", ptr);

{% endhighlight %} 

##函数和函数指针
在32-bit和64-bit中函数调用最明显的区别就是对可变参数函数的调用。

在64位运行时函数调用的处理方式不同于在32位运行时功能。关键的区别是，函数调用与可变参数的原型使用的指令不同的顺序来读取它们的参数比利用参数固定的列表功能。清单2-12显示了原型两种功能。第一个函数（固定功能）始终以一副整数。所述第二函数采用可变数目的参数（但至少两个）。在32位运行时，两个函数的清单2-12使用类似的指令序列，以读出的参数数据中调用。在64位运行时，这两个功能使用的约定是完全不同的编译。
因为调用约定是多大，在64位运行时更精确，你需要确保一个功能总是正确调用，使被叫方总是发现来电者提供的参数。


##不要直接访问Objective-C 指针
在64-bit运行时中，如果我们的代码直接访问了object->isa的话将会引起错误。因为在64-bit中*isa*已经不仅仅是一个指针了，他除了包含指针之外还用剩余的bits存储了一些其他的运行时信息。这个优化能同时改善内存的使用和应用的性能
当然，我们可以使用下列方法达到同样的目的
{% highlight objective-c%}
#include <inttypes.h>
class
object_getClass 
object_setClass
{% endhighlight %} 

`重要：由于这个错误不会在模拟器上重现，所以我们需要用真机进行测试`

###Use the Built-in Synchronization Primitives

Sometimes, apps implement their own synchronization primitives to improve performance. The iOS runtime environment provides a full complement of primitives that are optimized and correct for each CPU on which they are running. This runtime library is updated as new architectural features are added. Existing 32-bit apps that rely on the runtime library automatically take advantage of new CPU features introduced in the 64-bit world. If you implemented custom primitives in a previous version of your app, now you might be relying on instructions or paths that are orders of magnitude slower than the built-in primitives. Instead, always use the built-in primitives.



##避免以硬编码的形式获取内存分页大小
大多数应用都不需要知道内存分页大小信息，不排除有一些应用会用分配缓存，所以这里有必要强调一下。从iOS7开始，32-bit和64-bit的内存分页大小就已经完全不同了，所以我么一定要确保在用到内存分页大小的时候调用*getpagesize()*函数来获取正确的分页大小



##确保你的程序在32-bit下运行正常

目前，编写一个可在64位运行时环境下运行的APP同时也应该支持32位。那么应该保证我们的应用在任一环境下运行良好。通常来说也就是设计一个可同时在两种环境下运行良好的结构。但有时也需要针对不同的运行环境编写特殊的解决方案。
例如，你或许倾向于在整个代码中定义64位整型；因为不仅两种运行时环境都支持64位整型，而且仅使用一个单一的整型简化了APP的设计。但如果你使用全篇使用64位整型，你的应用将会在32位运行时环境下运行更慢一些。一个64位的处理器运行64位整数运算的速度和运行32位整数运算一样，但32位的处理器运行64位的运算就要慢很多。同时，在两种环境下，变量可能会占用更多一些的内存。所以我们应该根据要处理的数据范围选择一个合适的整数类型。如果是一个32的整数运算，就定义一个32位的整数。





















