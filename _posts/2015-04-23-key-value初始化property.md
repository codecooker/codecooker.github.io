---
layout: post
title: key-value初始化property
category: iOS
tags: [iOS]
---

最近在做项目的过程中，有用到以*key-value*的形式初始化对象，常规做法就是实现一份儿解析根据不同的映射初始化不同的变量。但由于我们需要让使用方更大的简化调用方式，所以引入了该方法来实现这些功能。其实就是用了一些*oc*语言的特性来处理这些事情。

###为什么提出
由于在处理一些特有逻辑的时候，我们需要用一些*key-value*的形式或者配置文件来初始化对象，而传统的*encode*和*decode*方法则需要在类里边根据自己的实现来实现一份儿代码。所以我们需要一个独立的模块来出来该类问题。要么是不方便，要么是代码过于分散不便管理

###从 0 到 1
我们可以定制一个映射规则，这篇文章内的映射规则是这样的
>以*array*的形式传入要初始化的参数，参数的item为一个*map*类型。*map*的定义格式为*key*为对应对象内的*property*,*value*则为我们要初始化成的值
当然为了保险起见，我们也可以在找不到*property*的情况下查找类是否有*key*对应的成员变量，然后对其进行初始化

###接下来
我们先来看看如下的代码
{% highlight objective-c%}
//根据propery名称获取类对应的propery的指针
objc_property_t property = class_getProperty([self class] , propertyName.UTF8String);
//根据property来获取property属性字符串
NSString *propertyAttributesString = [NSString stringWithUTF8String:property_getAttributes(property)];
{% endhighlight %} 

其中objc_property_t类型是一个系统类型的指针，对开发者是不可见的，但是apple为我们提供了一系列的方法来处理它，我们来看看都有哪些方法可以处理它，我在注释里做了一个简单的介绍
{% highlight objective-c%}
//获取objc_property_t指针指向的property名称
OBJC_EXPORT const char *property_getName(objc_property_t property) 
     __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);
//获取objc_property_t指针指向的property的属性描述，后边会说明格式
OBJC_EXPORT const char *property_getAttributes(objc_property_t property) 
     __OSX_AVAILABLE_STARTING(__MAC_10_5, __IPHONE_2_0);
//获取objc_property_t指针指向的property的属性描述，返回值为objc_property_attribute_t类型，objc_property_attribute_t是一个结构体，参数outCount表示需要返回多少个objc_property_attribute_t对象
OBJC_EXPORT objc_property_attribute_t *property_copyAttributeList(objc_property_t property, unsigned int *outCount)
     __OSX_AVAILABLE_STARTING(__MAC_10_7, __IPHONE_4_3);
//copy属性
OBJC_EXPORT char *property_copyAttributeValue(objc_property_t property, const char *attributeName)
     __OSX_AVAILABLE_STARTING(__MAC_10_7, __IPHONE_4_3);
{% endhighlight %} 

其中objc_property_attribute_t的定义如下
{% highlight objective-c%}
/// Defines a property attribute
typedef struct {
    const char *name;           /**< The name of the attribute */
    const char *value;          /**< The value of the attribute (usually empty) */
} objc_property_attribute_t;
{% endhighlight %} 

这里我们用到了<kp>property_getAttributes</kp>方法，所以将<kp>property_getAttributes</kp>方法的返回值着重说一下，当我们调用该方法时，它的返回值一般是这个样式的
{% highlight objective-c%}
T@"NSString",C,N,V_title
//分组后是这样的
T@"NSString",           //表示类型，是一个OC对象，具体为NSString的对象
C,                                //copy
N,                                //nonatomic
V_title                         //对应成员变量_title，即在.m 里做了@synthesize title = _title_
{% endhighlight %} 

当我们重写了setter和getter方法后，我们还会看到*G<name>*和*S<name>*，尖括号内为我们自定义的setter或getter方法
如*ScustomSetter:*表示我们重写了*setter*方法为customSetter:

###动手实现
有了上述的基础知识作为铺垫后，我们就可以开始实现我们的初始化模块了  
1. 定义映射规则  
2. 根据key找到类中的propery（成员变量下次说）  
3. 根据propery获取到property属性，获取正确的setter方法和数据类型  
4. 调用perform或者msg_send方法进行复制  

具体实现代码，可以参考下边实现
{% highlight objective-c%}
- (BOOL)setPropertyWithVariable : (id)variable property : (NSString *)propertyName
{
    objc_property_t property = class_getProperty([self class] , propertyName.UTF8String);
    if (property != NULL) {
        __block NSString *propertySetterName = [NSString stringWithFormat:@"set%@:",[NSString stringWithFormat:@"%@%@",[[propertyName substringToIndex:1]uppercaseString],propertyName.length > 1 ? [propertyName substringFromIndex:1] : @""]];
        __block NSString *propertyType = nil;
        NSString *propertyAttributesString = [NSString stringWithUTF8String:property_getAttributes(property)];
        NSArray *propertyAttributes = [propertyAttributesString componentsSeparatedByString:@","];
        [propertyAttributes enumerateObjectsUsingBlock:^(id aObj, NSUInteger idx, BOOL *aStop) {
            if ([aObj hasPrefix:@"S"]) {
                propertySetterName = [aObj substringFromIndex:1];
            }else if ([aObj hasPrefix:@"T"]){
                NSArray *parts = [aObj componentsSeparatedByString:@","];
                propertyType = [parts objectAtIndex:parts.count - 1];
            }
        }];
        SEL setSelector = sel_registerName(propertySetterName.UTF8String);
        if ([propertyType isEqualToString:@"Tc"]) {
            if (strlen([variable UTF8String])) {
                void (*objc_msgSend_char)(id self, SEL _cmd, char c) = (void*)objc_msgSend;
                objc_msgSend_char(self, setSelector,[variable UTF8String][0]);
            }
        }else if ([propertyType isEqualToString:@"Td"]){
            void (*objc_msgSend_double)(id self, SEL _cmd, double c) = (void*)objc_msgSend;
            objc_msgSend_double(self, setSelector,[variable doubleValue]);
        }else if ([propertyType isEqualToString:@"VenumDefault"]){
            void (*objc_msgSend_integer)(id self, SEL _cmd, NSInteger c) = (void*)objc_msgSend;
            objc_msgSend_integer(self,setSelector,[variable integerValue]);
        }else if ([propertyType isEqualToString:@"Tf"]){
            void (*objc_msgSend_float)(id self, SEL _cmd, float c) = (void*)objc_msgSend;
            objc_msgSend_float(self, setSelector,[variable floatValue]);
        }else if ([propertyType isEqualToString:@"Tl"]){
            void (*objc_msgSend_long)(id self, SEL _cmd, long c) = (void*)objc_msgSend;
            objc_msgSend_long(self, setSelector,[variable longValue]);
        }else if ([propertyType isEqualToString:@"Ts"]){
            void (*objc_msgSend_short)(id self, SEL _cmd, short c) = (void*)objc_msgSend;
            objc_msgSend_short(self, setSelector,[variable shortValue]);
        }else if([propertyType isEqualToString:@"Ti"]){
            void (*objc_msgSend_integerValue)(id self, SEL _cmd, NSInteger c) = (void*)objc_msgSend;
            objc_msgSend_integerValue(self, setSelector,[variable integerValue]);
        }else{
            void (*objc_msgSend_id)(id self, SEL _cmd, id c) = (void*)objc_msgSend;
            objc_msgSend_id(self, setSelector,variable);
        }
        return YES;
    }
    return NO;
}
{% endhighlight %} 

上述代码做一些说明，我们调用了objc_msgSend方法来调用类或者对象的*set*方法，在64-bit运行环境中，我们可以了解到所有的方法必须有明确的定义，所以xcode6以后做了一个限制，不可直接调用该方法。具体可以参见该方法的定义。那我们为了调用该方法该怎么做呢。这里有两种方案：  
>1. 将objc_msgSend方法转换成有明确定义的函数指针，然后再进行调用
2. 在build setting 中关闭xcode的objc_msgSend方法直接调用检查

推荐使用第一种方法，因为第二种方法违背了64-bit的那个规定，会存在潜在风险


###*property_getAttributes*的一些说明

![property_getAttributes说明]({{ site.url }}/images/key-value形式初始化对象/Snip20150424_1.png)







