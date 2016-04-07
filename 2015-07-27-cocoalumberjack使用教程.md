---
layout: post
title: "CocoaLumberjack使用教程"
description: ""
category: ios开发
tags: []
author: 饭小团
--- 

###CocoaLumberjack使用教程

对于ios和mac，CocoaLumberjack 是一个快速，简单，又是个强大，灵活的日志框架。
这篇教程会告诉你如何简单的使用它，并且在一个颜色插件的帮助下可以让你的输出日志有层次和多样化。
实际效果如下图所示：

![Alt text](../attachment/fanshen/CocoaLumberjack.png)


* 通过 CocoaPods 安装
* Podfile 放进 pod 'CocoaLumberjack'
* update

打开项目appDelegate.m里

	#import <CocoaLumberjack/CocoaLumberjack.h>
	
我们要全局使用它。

一搬情况下，我们需要配置它在applicationDidFinishLaunching方法里。

	[DDLog addLogger:[DDASLLogger sharedInstance]];
	[DDLog addLogger:[DDTTYLogger sharedInstance]];
	
DDASLLogger 会把log信息输出到Console.app。DDTTYLogger 会把log信息输出到 Xcode console
现在你可以用DDLog 替换掉 NSLog 函数，如下所示：

	// Convert from this:
	NSLog(@"Broken sprocket detected!");
	NSLog(@"User selected file:%@ withSize:%u", filePath, fileSize);

	// To this:
	DDLogError(@"Broken sprocket detected!");
	DDLogVerbose(@"User selected file:%@ withSize:%u", filePath, fileSize);
	// DDLogError和DDLogVerbose 是log级别，稍后会讲到
	
不过，你现在还不能run起来。因为还需要定义一个log级别才行。你可以加入这句话：

	static const DDLogLevel ddLogLevel = DDLogLevelVerbose;

它定义了一个log级别为DDLogLevelVerbose。这个框架为我们定义了多个log级别，我们来看看这些级别都是些什么。
目前框架内部有5个级别：

* DDLogError
* DDLogWarn
* DDLogInfo
* DDLogDebug
* DDLogVerbose

解释：

* 如果你定义log级别是DDLogLevelError，  那么你只能看到Error的状态。
* 如果你定义log级别是DDLogLevelWarn，   那么你能看到Error 和 Warn 的状态。
* 如果你定义log级别是DDLogLevelInfo，   那么你能看到Error, Warn 和 Info的状态。
* 如果你定义log级别是DDLogLevelDebug，  那么你能看到Error, Warn, Info 和 Debug 的状态。
* 如果你定义log级别是DDLogLevelVerbose，那么你能看到所有DDLog的状态。
* 如果你定义log级别是DDLogLevelOff，    那么你将看不到所有DDLog的状态。

就是说:

如果你定义log级别是DDLogLevelError，那控制台只会打印DDLogError()里的内容.

如果你定义log级别是DDLogLevelWarn，那控制台会打印DDLogError()和DDLogWarn()里的内容.

不过我觉得你肯定不会在全局里定义某个级别，那样他会在所有地方只打印某些DDLog。你完全可以在每个文件，或每个功能前定义级别，使得输出内容更具灵活性。

也就是说，你可以自己定义在哪个地方输出什么样的信息，是error，还是warn，或者是所有info。你都可以自行决定。

###让log具有自己的色彩

控制台只输出一两种颜色实在太糟了，完全找不见需要的东西。我们来安装个xcode插件与CocoaLumberjack，让它丰富一些吧。
直接从这里[XcodeColors 链接](https://github.com/robbiehanson/XcodeColors)下载XcodeColors，run一次后重启xocde即可。

####解决：如果你的控制台没有任何颜色

* Product -> Scheme -> Edit Scheme
* 选择左边的 "Run", 然后选择右边的 "Arguments" 
* 下面有个  Environment Variable，添加一个Name为 XcodeColors ，Value为 YES。

![Alt text](../attachment/fanshen/CocoaLumberjack2.png)


把原来的初始化addLogger代码替换掉

	// Standard lumberjack initialization
	[DDLog addLogger:[DDTTYLogger sharedInstance]];

	// And we also enable colors
	[[DDTTYLogger sharedInstance] setColorsEnabled:YES];


默认情况下的颜色是这样：

* DDLogError : Prints in red
* DDLogWarn : Prints in orange

你在DDLogError时，输出的颜色是红的。

你在DDLogWarn时，输出的颜色是橘黄的。

你也可以自定义颜色，对于不同级别的log。

	NSColor *pink = [NSColor colorWithCalibratedRed:(255/255.0) green:(58/255.0) blue:(159/255.0) alpha:1.0];

	[[DDTTYLogger sharedInstance] setForegroundColor:pink backgroundColor:nil forFlag:DDLogFlagInfo];

	DDLogInfo(@"Warming up printer"); // Prints in Pink !

现在打印出的字体颜色是粉色。

看见那个backgroundColor:了吗？ 它可以设置背景色~



####Something else

* 如果你想把DDLogLevelInfo级别的内容输出到Console.app中，DDLogLevelDebug级别输出到xcode的控制台中。可以使用这个函数：
	
		[DDLog addLogger:[DDASLLogger sharedInstance] withLevel:DDLogLevelInfo];
		[DDLog addLogger:[DDTTYLogger sharedInstance] withLevel:DDLogLevelDebug];

* 如果你想把你的log信息放在一个文件里，在某个时候把他上传到你的服务器

		DDFileLogger *fileLogger = [[DDFileLogger alloc] init];
		fileLogger.rollingFrequency = 60 * 60 * 24; // 24 hour rolling
		fileLogger.logFileManager.maximumNumberOfLogFiles = 7;
		[DDLog addLogger:fileLogger];  

具体的操作请看这里[LogFileManagement.md](https://github.com/CocoaLumberjack/CocoaLumberjack/blob/master/Documentation/LogFileManagement.md)


