---
layout: post
title: iOS测量代码执行时间
---

iOS准确的测量系统执行时间:在iOS的开发过程中，经常会为了满足严苛的用户体验要求，需要对某些功能和模块的执行时间进行控制，那么问题来了，如何高效的检测代码的执行时间



####涉及的API#
先来认识几个API，他们被包含在`<mach/mach_time.h>`中  
在使用前我们需要先  
`#import <mach/mach_time.h>`  
进入mach_time.h文件，我们可以看到如下内容 
<pre>
struct mach_timebase_info {
    uint32_t    numer;
    uint32_t    denom;
};
typedef struct mach_timebase_info   *mach_timebase_info_t;
typedef struct mach_timebase_info   mach_timebase_info_data_t;

__BEGIN_DECLS
kern_return_t       mach_timebase_info(mach_timebase_info_t    info);
kern_return_t       mach_wait_until(uint64_t        deadline);
uint64_t            mach_absolute_time(void);
uint64_t            mach_approximate_time(void);
__END_DECLS
</pre>
首先定义了一个mach_timebase_info的结构体，用来存储CPU的一些执行信息,改结构体包含两个成员  
1. numer 直译为分子  
2. denom直译为分母
具体用法后边会有所描述，各位看官可以先继续往下看  
`mach_timebase_info(mach_timebase_info_t    info)`方法用来初始化一个mach_timebase_info的对象  
`mach_wait_until(uint64_t        deadline);`用来挂起线程等待deadline时唤醒  
`mach_absolute_time(void)`获取当前的绝对时间
`mach_approximate_time(void)`获取当前的大概时间，如果可以忍受一点误差就可以选用该方法，后边会有对比  
####如何使用#
<pre>
mach_timebase_info_data_t timebase_info;
mach_timebase_info(&timebase_info);
uint64_t startTime = mach_absolute_time();
for (int i = 0; i < 10000; ++i) {
    NSLog(@"Hello world");
}
uint64_t endTime = mach_absolute_time();
NSLog(@"消耗时间%lld",endTime - startTime);
</pre>
结果显示为：*消耗时间4822588154*  
当然这个时间不是实际消耗时间，而是CPU的运行周期数，我们要获取实际运行时间，还需要做一些处理  
#####numer 、denom怎么用#
举个例子吧，假设numer = 100；denom = 10；则表示一个CPU时钟周期为100/10=10（单位为纳秒）  
所以要得到准确时间，我们需要添加如下代码：  
<pre>
        uint64_t tureTime = (endTime - startTime) * timebase_info.numer / timebase_info.denom;
        NSLog(@"消耗时间%lld",tureTime);
</pre>
结果显示为：*消耗时间5087630559*  ,注意这里单位为***纳秒***，即正确时间为5.08秒.    
#####`mach_approximate_time的返回结果`#
结果显示为:*消耗时间6939457729*,即6.9秒

