---
layout: post
title: About 64-Bit Cocoa Touch Apps
category: iOS
tags: [iOS,Build,64-bit]
---



随着桌面操作系统由32-bit到64-bit的过渡，64-bit的app的出现也在被OS的发展驱动着。现在，iOS的系统架构和桌面系统的架构也有很多相同的地方。从iOS7的A7处理器开始，我们就可以build支持64-bit处理器的app了。当app支持了64bit后在同样的机型上可以获取更高的性能  
##简介
 apple 的A7处理器支持两个不同的指令集。一个是由apple早前处理器支持的32bit ARM指令集；另一个则是一个全新的64-bit ARM架构处理器。64-bit架构最重要的改进不局限于支持更大的地址空间，更是提供一个全新的The 64-bit architecture supports a new and streamlined instruction set that supports twice as many integer and floating-point registers。LLVM比编译器可以将64-bit架构的优势发挥到极致。64-bit apps 可以一次处理更大的数据来改善性能。所以虽然我们的32-bit应用在A7处理器上已经比之前的处理器上运行快了很多，但是为了获得更好的性能和体验，我们还是需要对我们的应用进行64-bit的适配（升级）  
 在64-bit的iOS应用中，指针占64bits，一些整数类型也由之前的32bits变成了64bits。很多系统framework包含的数据类型也随之改变。这些改变意味着64-bit的应用将会比32-bit的应用在运行时占用更大的内存空间，如果我们不做进一步的内存管理和优化，这些内存的消耗量将会损给我们应用的性能和体验  
 我们在64-bit设备上运行iOS系统时，iOS包含独立的32-bit和64-bit系统框架。当设备上的所有应用都是64-bit时，iOS则不需要加载32-bit的系统framework，这样就确保了系统可以以很低的开销来启动并运行应用。反之则需要加载两套系统framework，耗费更多的资源，性能随之变差  

###升级iOS7后将应用适配到64-bit
Xcode5.0.1之后的版本支持构建32-bit和64-bit的二进制包。如果要同事包含32-bit和64-bit则需要最低支持版本为iOS5.1.1 64-bit的app需要运行在最低iOS7.0.3的系统版本。如果我们是对已有的应用进行适配，则需要先将app适配iOS7然后在适配64-bit处理器  
iOS上的64-bit架构和OS X上的基本一致，这样我们就可以很容易的开发一套代码方便的运行在两个系统中  
开始构建64-bit的app时，我们需要修复一些和64-bit有关的warning，例如：  
* 确保所以调用的方法都存在实现  
* 避免就爱那个64-bit的数据类型在的隐式的赋值给32-bit数据类型  
* 确保数值计算在64-bit中也时同样正确的  
* 创建在32-bit、64-bit中数据对齐一致的数据结构  
 
###在64-Bit设备上测试app
iOS模拟器提供32-bit和64-bit的设备。当我们完成64-适配后，我们可以用模拟器对app进行测试，在测试阶段，模拟器是我们最好的选择。但是切记在我们正式发布引用之前我们必须在真机上进行测试，因为一些运行时的改变只有当我们的app运行在真机上才能定位和查明  
当我们的app已经完全的适配完成后，再用Instruments来测量和优化性能和内存的使用即可


