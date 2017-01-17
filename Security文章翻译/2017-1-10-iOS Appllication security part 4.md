---  
layout: post
title: "iOS Appllication security part 4 runtime analysis using cycript weather app "
description: ""
category: ios
tags: [Security]
author: 饭小团
---   


#iOS Appllication security part 4 
##runtime analysis using cycript weather app

本章我们要学习如何使用Cycript工具来分析和修改iOSapp的运行时。  

Cycript:  
Cycript是一个js解释器，他可以理解Objective-C语法，这意味着在一个特殊的命令行下我们既能写Objective-C语言，又能写js语言。他可以hook住一个运行时进程并且帮助我们在运行时修改很多东西。  
以下是一些使用Cycript的优势：  

	1.可以hook进一个运行进程，找出所有正在使用的class名字，如：viewController，网络，第三方库。甚至Application的delegate。  
	2.对于一个特殊的class，如ViewController，App delegate或其他class，我们仍可以找出所有已使用的函数名。  
	3.可以找见所有实例变量名和他们在运行时的值。  
	4.在运行时修改实例变量的值。  
	5.使用Method Swizzling替换某一块代码。  
	6.在运行时任意调用任何函数。  

##安装Cycript  
直接在Cydia里搜索Cycript，安装即可。  

##使用Cycript在运行时修改  
这里使用一个天气app(随便找一个就行)来做测试。  
打开这个天气app，确保是始终运行在前台。如果运行在后台他就会暂停，你就啥也不能做了。运行在前台我们就直接hook他的进程。首先先获取他的进程。   
ps aux | grep "weather"  
找见进程后： cycript -p XXX(进程号)  
如图就证明hook成功了。
![hook](http://highaltitudehacks.com/images/posts/ios4/5x.png)  
使用Objective-C语言来获得[UIApplication sharedApplication]  
![](http://highaltitudehacks.com/images/posts/ios4/5.png)
也可以使用变量a来引用他。  
![](http://highaltitudehacks.com/images/posts/ios4/6.png)  

为了可以找见这个application的delegate，直接使用a.delegate就可以找见  
![](http://highaltitudehacks.com/images/posts/ios4/8.png)  

尝试隐藏或显示状态栏  
![](http://highaltitudehacks.com/images/posts/ios4/9.png)  

修改 badge count 
![badge count](http://highaltitudehacks.com/images/posts/ios4/9x.png)  

假如我们要找见current view controller，我们需先找见keyWindow。keyWindow就是接受当前用户点击事件的window。如果想找见所有的Windows，使用windows命令。  
![windows](http://highaltitudehacks.com/images/posts/ios4/10.png)
为了找见keyWindow，你可以这样  
![keyWindow](http://highaltitudehacks.com/images/posts/ios4/11.png)
使用keyWindow的属性找见rootViewController  
![rootViewController](http://highaltitudehacks.com/images/posts/ios4/14.png)   

##Conclusion  
基本就这样，比较简单的入门~  
下一章我们会寻找指定类的所有函数并且修改他的实现。我们也会修改指定class的实例变量值。  