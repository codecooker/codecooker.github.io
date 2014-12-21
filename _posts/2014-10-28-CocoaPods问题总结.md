---
layout: post
title: CocoaPods问题总结
---
在项目接入**cocoapods**后，打包机器打包方面暴漏和很多问题。总会莫名其妙的发生错误，让大家手足无措。  
这里我来总结下常见的一些问题的处理方案，方便大家在日后的项目测试中做到有的放矢。  



####1.找不到相应的库#
找不到相应的库的问题，我碰到的有两个原因  
1.打包机器没有添加**framework**对应的源（如何添加源，请继续往下看）  
2.本地**cocoapods**的版本与**打包机器**上**cocoapods**的版本不一致，这个问题通常是由于本地的版本高于MTL版本造成的

####2.找不到相应的**target**#
由于工程的迁移，或者开发人员的误操作，会造成本地**target**的丢失。这样在命令行打包过程中，会暴找不到**target**的错误，这时我们只需要将针对与个人的**target**设置成**shared**即可，如下:  
![设置target为shared]({{ site.url }}/images/{{page.title}}/Snip20141028_7.png)  
***重要:这里需要注意的一点是我们的更改怎么提交到git上，如果大家这样操作后会发现git并没有跟踪到任何变化，我们无所提交。其实我们只需要在git托管的目录下找到.gitignore文件(这是一个隐藏文件，如果不清楚这个文件是做什么的，自行恶补),彩票的文件内容如下***

<pre>
.DS_Store

build/*
docs/*
.svn/

# Xcode
*.pbxuser
*.mode1v3
*.mode2v3
*.perspectivev3
*.xcuserstate
*.xcworkspace
xcuserdata
*.xcscheme
*.xccheckout
profile
*.moved-aside
DerivedData
*.hmap
*.ipa
Pods
.idea
contents.xcworkspacedata
</pre>  
***我们需要暂时屏蔽对*.xcscheme文件的过滤，然后就能看到更改，接下来commit->push**

####3.本地升级cocoapods后与MTL版本不一致#
这个问题通常**不会直接抛出**是由于本地**cocoapods**版本与打包机器**cocoapods**版本不一致的错误异常  
通常的展现方式会以在编译的时候**找不到某个库的错误**暴露出来，如:  
<pre>
error: 'XXXFramework/XXXFramework.h' file not found
    #import "XXXFramework/XXXFramework.h"
            ^
1 error generated.
</pre>
在确定了已经添加相应的源且获得了相应的权限后,碰到类似诡异的问题,可以翻看*log*文件的前面,查看是否有生成**podfile.lock**的**cocoapods**版本号和打包机器的**cocoapods**版本号不一致的警告。  
如下:  
`[!] The version of CocoaPods used to generate the lockfile (0.34.2) is higher than the version of the current executable (0.33.1). Incompatibility issues may arise.`  
当看到如此的警告时我们可以按照如下步骤进行处理  
1.删除工程内的**Pods**文件夹  
2.删除工程内的**xcworkspace**的文件  
3.删除**Podfile.lock**文件  
4.打开**xcodeproj**,注意不是**xcworkspace**,在左边树形结构文件处删除**Pods**文件夹  

![pods文件夹图片]({{ site.url }}/images/{{page.title}}/Snip20141028_3.png)  
5.删除功能内对**Pod**产生**target**的引用  
6.删除**Pod**产生的一些配置信息如图  
![pods文件夹图片]({{ site.url }}/images/{{page.title}}/Snip20141028_5.png)
图中的**第二项**和**最后一项**  
7.提交代码打包

####PS:其他涉及到的问题，大家熟悉cocoapods的话，应该都能搞定。如果还诡异问题也可以和我沟通，相信以后大家都能搞定摩天轮打包失败的问题了，哈哈！#

####注:cocoapods常见低级问题解答#
######1.如何添加源#
`pod repo add [source_name] [source_path];`  

其中source_name指为源起的名称，可以根据自己的喜好，但是不要和本地源重名。source_path为我们真是源的一个git地址，注意源的git权限通常需要设置为**public**  
升级cocoapods到最新版后，我们也可以这样使用  
在podfile里添加如下代码指定该podfile内涉及的库从哪些源里获取  

`source [source_path];`

######2.是否需要提交Podfile.lock#
这是CocoaPods创建的最重要的文件之一。它记录了需要被安装的pod的每个已安装的版本。如果你想知道已安装的pod是哪个版本，可以查看这个文件。推荐将Podfile.lock文件加入到版本控制中，这有助于整个团队的一致性。 

######3.Manifest.lock是干什么的,为什么一致报错#
这是每次运行pod install时创建的Podfile.lock文件的副本。如果大家遇到过这个错误：
  
`diff: /../Podfile.lock: No such file or directory diff: /Manifest.lock: No such file or directory error: The sandbox is not in sync with the Podfile.lock. Run 'pod install' or update your CocoaPods installation.`  

由于Pods所在的目录并不总在版本控制之下，这样可以保证开发者运行app之前都能更新他们的pods，否则app可能会crash，或者在一些不太明显的地方编译失败.

