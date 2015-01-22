---
layout: post
title: Linux编译Objective-C 
category: objective-c
tags: [objective-c,linux,ubuntu]
---

首先安装必须的包： 
(提供Ubuntu下安装方法)  
1.gobjc
{% highlight sh%}
sudo apt-get install gobjc
{% endhighlight %} 
2.gnustep
{% highlight sh%}
sudo apt-get install gnustep
{% endhighlight %} 
3.gnustep-devel
{% highlight sh%}
sudo apt-get install gnustep-devel  
{% endhighlight %} 
安装完成后可用如下代码测试环境：

{% highlight objective-c%}
//main.m
#import <Foundation/Foundation.h>
int main (int argc, const char * argv[])
{
        NSLog (@"hello Objective-C\n");
        return 0;
}
{% endhighlight %} 

编译命令为：
{% highlight sh%}
gcc `gnustep-config --objc-flags` -lgnustep-base main.m -o main
{% endhighlight %} 