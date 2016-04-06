---
layout: post
title: iOS逆向工程-dumpdecrypted砸壳 
category: objective-c
tags: [objective-c,iOS]
---



近期想打开大门学习下同行们的iOS开发，于是想到了可以针对一些有特色的应用进行逆向工程，然后着重学习下该应用的设计架构。  
由于自己也是初学者，下来就把自己躺过的坑记录在这里方便后续查阅。

#### 逆向工程需要的工具
1. dumpdecrypted  
dumpdecrypted是一个砸壳工具。因为appstore安装包apple会为应用加上一层壳，所以在逆向之前我们需要先对应用进行砸壳。官方对dumpdecrypted描述是
>Dumps decrypted iPhone Applications to a file  
详细信息可以参考[dumpdecrypted](http://example.com/ "Title")
2. class-dump
为什么是class-dump

>It’s a great tool for the curious. You can look at the design of closed source applications, frameworks, and bundles. Watch the interfaces evolve between releases. Experiment with private frameworks, or see what private goodies are hiding in the AppKit. Learn about the plugin API lurking in Mail.app.  

**class-dump又是什么呢？**  
class-dump是用于解析Mach-O文件中存储的OC运行时信息的。他能生成类的声明、分类、协议。和otool -ov类似，但是由于class-dump的结果是以OC代码展示的，所以有很强的可读性。
3. 一台越狱设备(重要)  
iOS为了保证安全性，所有的应用都是在沙盒中运行。所以要想完成逆向操作，我们需要更高级的权限。没有越狱设备那就无法展开学习和研究工作。

#### 准备工作
1. 编译砸壳工具dumpdecrypted.dylib  
下载dumpdecrypted的源码[dumpdecrypted下载](https://github.com/stefanesser/dumpdecrypted "Title")，打开Makefile文件，如下
{% highlight sh%}
#获取gcc编译器的路径
GCC_BIN=`xcrun --sdk iphoneos --find gcc`      
# 编译出得dumpdecrypted支持的CPU架构，可根据我们设备的CPU架构自行增减，一般情况下不需要修改
GCC_UNIVERSAL=$(GCC_BASE) -arch armv7 -arch armv7s -arch arm64 
#获取SDK的路径
SDK=`xcrun --sdk iphoneos --show-sdk-path`

CFLAGS = 
GCC_BASE = $(GCC_BIN) -Os $(CFLAGS) -Wimplicit -isysroot $(SDK) -F$(SDK)/System/Library/Frameworks -F$(SDK)/System/Library/PrivateFrameworks

all: dumpdecrypted.dylib

dumpdecrypted.dylib: dumpdecrypted.o 
    $(GCC_UNIVERSAL) -dynamiclib -o $@ $^

%.o: %.c
    $(GCC_UNIVERSAL) -c -o $@ $< 

clean:
    rm -f *.o dumpdecrypted.dylib
{% endhighlight %} 
我对上述几个可能需要改动的地方添加了部分注释，可供参考   
2. 安装Class-dump  
可以从[Class-dump下载](http://stevenygard.com/projects/class-dump/ "Title")下载源码和dmg包，推荐使用dmg，很省事儿。  
3. 确保设备上安装了openSSH，使能链接到设备  
4. 确保安装了cycript,关于cycript后续会专门总结，大家只需要知道cycript  
>cycript允许开发者修改在iOS和Mac OS X上运行的应用

#### 开始工作
1. 将我们编译好的dumpdecrypted.dylib部署copy到手机  
`scp $myDumpdecryptedDocument/dumpdecrypted.dylib root@xxx.xxx.xxx.xxx:/tmp`
 将上述命令中得$myDumpdecryptedDocument替换成dumpdecrypted.dylib所在的目录，xxx.xxx.xxx.xxx替换成设备的IP地址
 2. 获取应用的路径和PID
{% highlight sh%}
ssh root@xxx.xxx.xxx.xxx #ssh登录设备
ps aux; //输出正在运行的进程
{% endhighlight %} 
如图显示
![PID结果](/images/iOS逆向工程-dumpdecrypted砸壳/1.png)
图中PID即表示对应进程的id，command即表示改该应该的app路径  
3. 砸壳  
由于iOS沙盒的限制，所以我们需要将dumpdecrypted.dylib移动到对应app的documents目录下，然后进行砸壳。在iOS8之前，documents目录在command所在目录的上层，而在iOS8中这个路径发生了变化，我们可以借助cycript来找到app对应的documents目录。
执行
{% highlight sh%}
cycript -p pid
{% endhighlight %} 
其中pid表示要砸壳应用对应的pid，进入cycript 命令模式，指向如下代码
{% highlight objective-c%}
[[NSFileManager defaultManager] URLsForDirectory:NSDocumentDirectory inDomains:NSUserDomainMask][0]
{% endhighlight %} 
输出结果即为app对应的documents目录，将dumpdecrypted.dylib移动到该documents中
执行
{% highlight sh%}
DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib /var/mobile/Applications/XXXXXX/app.app/app
{% endhighlight %} 

参数为上述的command，执行后会看到如下结果表示成功
{% highlight sh%}
[+] detected 32bit ARM binary in memory.
[+] offset to cryptid found: @0x4abc(from 0x4000) = abc
[+] Found encrypted data at address 00004000 of length 23347200 bytes - type 1.
[+] Opening /private/var/mobile/Applications/71C51582-9552-48DD-9811-B99C991F55BE/MicroMessenger.app/MicroMessenger for reading.
[+] Reading header
[+] Detecting header type
[+] Executable is a plain MACH-O image
[+] Opening MicroMessenger.decrypted for writing.
[+] Copying the not encrypted start of the file
[+] Dumping the decrypted data into the file
[+] Copying the not encrypted remainder of the file
[+] Setting the LC_ENCRYPTION_INFO->cryptid to 0 at offset abc
[+] Closing original file
[+] Closing dump file
{% endhighlight %} 
这时在当前目录下会生成一个appname.decrypted,这个就是砸过壳的应用了。

4. 生成文件
{% highlight sh%}
class-dump --arch armv7 app.decrypted > app.m
{% endhighlight %} 

将生成的文件定向到当前目录的app.m文件中,打开文件就能看到内容了.

#### 一些补充

1. DYLD_INSERT_LIBRARIES是什么？  
是一个环境变量，在执行先先预加载该环境变量的内的动态库。

</br>
</br>

#### 参考资料

1. [https://github.com/stefanesser/dumpdecrypted](https://github.com/stefanesser/dumpdecrypted "dumpdecrypted")
2. [http://stevenygard.com/projects/class-dump/](http://stevenygard.com/projects/class-dump/ "class-dump")

