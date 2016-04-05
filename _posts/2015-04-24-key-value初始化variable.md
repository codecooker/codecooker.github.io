---
layout: post
title: key-value初始化instanceVariable
category: iOS
tags: [iOS]
---

接上一篇来介绍下，如何用key-value的形式初始化成员变量  

背景接上篇，就不在赘述，如果不了解映射规则的话，可以先看看上一篇  
[key-value初始化property](http://codecooker.cn/2015/04/key-value初始化property.html "key-value初始化property");

有个上一篇的铺垫，这篇将很好被接受，首先我们通过如下方法实例变量的名称获取到实例变量的指针*objc_ivar *Ivar;*
{% highlight objective-c%}
Ivar instanceVariable = class_getClassVariable([self class], variableName.UTF8String);
{% endhighlight %} 

同样*Ivar*类型为一个系统类型，我们不知道其内部实现，当然apple也同样为我们提供了方法来处理它，和上篇一样，先对处理它的方法做个描述

{% highlight objective-c%}
OBJC_EXPORT Ivar class_getClassVariable(Class cls, const char *name) 
     __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);

OBJC_EXPORT Ivar class_getInstanceVariable(Class cls, const char *name)
    __OSX_AVAILABLE_STARTING(__MAC_10_0, __IPHONE_2_0);

OBJC_EXPORT size_t class_getInstanceSize(Class cls) 
     __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);

OBJC_EXPORT Ivar *class_copyIvarList(Class cls, unsigned int *outCount) 
     __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);

OBJC_EXPORT void object_setIvar(id obj, Ivar ivar, id value) 
     __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);

OBJC_EXPORT Ivar object_setInstanceVariable(id obj, const char *name, void *value)
    __OSX_AVAILABLE_STARTING(__MAC_10_0, __IPHONE_2_0)
    OBJC_ARC_UNAVAILABLE;

OBJC_EXPORT id object_getIvar(id obj, Ivar ivar) 
     __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);

{% endhighlight %} 

这里我们要用到的就是*class_getInstanceVariable*方法、*class_getClassVariable*方法
、*object_setIvar*方法

###步骤
1. 获取成（类）员变量的指针Ivar instanceVariable
2. *object_setIvar*初始化

<kp>注意，如果是以字符串的形式传递参数，请在setIvar方法之前转换成合适的类型</kp>

实现可参加下面代码
{% highlight objective-c%}
- (void)setIvarWithVariable : (id)variable VariableName : (NSString *)variableName
{
    Ivar instanceVariable = class_getClassVariable([self class], variableName.UTF8String);
    if (instanceVariable) {
        NSString *type = [NSString stringWithUTF8String:ivar_getTypeEncoding(instanceVariable)];
        if ([type isEqualToString:@"c"]) {
            object_setIvar(self, instanceVariable, variableName);
        }else if ([type isEqualToString:@"i"]){
            object_setIvar(self, (__bridge Ivar)(@([variable intValue])), variableName);
        }else if ([type isEqualToString:@"s"]){
            object_setIvar(self, (__bridge Ivar)(@([variable shortValue])), variableName);
        }else if ([type isEqualToString:@"l"]){
            object_setIvar(self, (__bridge Ivar)(@([variable longValue])), variableName);
        }else if ([type isEqualToString:@"q"]){
            object_setIvar(self, (__bridge Ivar)(@([variable longLongValue])), variableName);
        }else if ([type isEqualToString:@"C"]){
            object_setIvar(self, (__bridge Ivar)(@([variable integerValue])), variableName);
        }else if ([type isEqualToString:@"S"]){
            object_setIvar(self, (__bridge Ivar)(@([variable unsignedShortValue])), variableName);
        }else if ([type isEqualToString:@"L"]){
            object_setIvar(self, (__bridge Ivar)(@([variable unsignedLongValue])), variableName);
        }else if ([type isEqualToString:@"Q"]){
            object_setIvar(self, (__bridge Ivar)(@([variable unsignedLongLongValue])), variableName);
        }else if ([type isEqualToString:@"f"]){
            object_setIvar(self, (__bridge Ivar)(@([variable floatValue])), variableName);
        }else if ([type isEqualToString:@"d"]){
            object_setIvar(self, (__bridge Ivar)(@([variable doubleValue])), variableName);
        }else if ([type isEqualToString:@"B"]){
            object_setIvar(self, (__bridge Ivar)(@([variable boolValue])), variableName);
        }else{
            object_setIvar(self, (__bridge Ivar)(variable), variableName);
        }
    }
}
{% endhighlight %} 

###关于类型定义的一些说明
![关于类型定义的一些说明]({{ site.res }}/images/key-value形式初始化对象/Snip20150504_1.png)

需要注意的是，oc中不支持long double类型，所以当double类型对待
