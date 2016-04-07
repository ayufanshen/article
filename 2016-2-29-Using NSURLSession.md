---
layout: post
title: "Using NSURLSession"
description: ""
category: 其它
tags: [Scrum]  
author: 饭小团  
---    
##Using NSURLSession

The NSURLSession class 和相关类提供了API用于通过HTTP下载内容。这个API提供了大量的代理方法用于支持认证并且给予你的app提供执行后台下载能力，即使你的app不在使用中。或者，你的app处于暂停状态。

为了使用 NSURLSession API，你的app会创建一系列的会话（Session），每一个都有对应的一组相关的数据传输任务（ transfer tasks）。举例，如果你在写一个web 浏览器，你的app可能在每个tab或window创建一个session。在每一个session，你的app添加了一系列的任务（Task）,每一个任务表示一个针对特定的URL的特定请求（request）,(and for any follow-on URLs if the original URL returned an HTTP redirect).

正如大部分网络APIs, NSURLSession API 是高度异步的，如果你用默认的，系统提供的代理。当你的app成功返回或者出错时，你必须提供完整的block处理返回数据。作为替补，如果你提供自定义代理对象，当数据从服务器返回时，Task对象会调用相应的代理方法。

	Note: Completion callbacks are primarily intended as an alternative to using a custom delegate. If you create a task using a method that takes a completion callback, the delegate methods for response and data delivery are not called.

The NSURLSession API 提供状态和进度条属性，附加传输信息到代理。支持取消，重新开始（恢复）和暂停任务。并且提供恢复已暂停，已取消，或者下载失败的地方从新开始。

##Understanding URL Session Concepts
task的行为在会话中依据三种方式：会话类型（Types of Sessions），任务类型（Types of Tasks），和当任务创建的时候无论app是否运行在前台。


##Types of Sessions
The NSURLSession API 支持三种会话方式，这个方式是由configuration object类型决定的，它用于创建session。  
 
 * Default sessions  类似于其他Foundation函数，用于下载URLs. 他们使用一个基于硬盘的持久化缓存方案和用用户的keychain存储认证
 
 * Ephemeral sessions 不会存储任何数据到硬盘；所有缓存，证书什么的都会放到RAM，并且绑定到session。因此，当你的app使session失效了，他们会被自动清除
 * Background sessions 类似于第一种Default sessions，除了有个单独的进程处理所有的数据传输，Background sessions 还有一些额外的限制。详细描述见 Background Transfer Considerations

##Types of Tasks
NSURLSession 提供三种任务类型：data tasks, download tasks, and upload tasks.

* Data tasks 发送和接受使用NSData objects。Data tasks 意在短小，经常和服务器交互的请求。Data tasks可以返回数据一次一小块，或者一次返回所有的通过一个完整的处理。

* Download tasks 以一个文件的形式得到数据，并且提供当app未运行时支持后台下载

* Upload tasks 以一个文件的形式发送数据，并且提供当app未运行时支持后台上传

##Background Transfer Considerations

NSURLSession class 支持后台传输，当你的app处于suspended，后台传输需要一个使用了background session configuration 对象的session。（as returned by a call to backgroundSessionConfiguration:）

对于background sessions，由于实际传输是由一个单独的进程执行并且由于重新启动你的app进程是相对昂贵的。由于以下限制，有一些功能是无法使用的：  
 
 * 会话必须提供代理对于事件传输（event delivery）。（对于上传和下载，代理行为和进程内的传输（in-process transfers）是一致的）
 * 仅支持HTTP HTTPS
 * 重定向总是跟随的（Redirects are always followed）
 * 只有upload tasks 是支持的。（程序退出上传会失败）
 * 如果background transfer 已经初始化了，当app处于后台时，configuration的discretionary 属性会当作ture
	
		Note:IOS8之前不支持后台session

在ios与OS X之间，你的app的行为在重启的时候回有一些不同 

in iOS，当后台传输任务完成的时候或需要证书时，如果你的app不在运行，ios会自动在后台重启app并且在UIApplicationDelegate调用application:handleEventsForBackgroundURLSession:completionHandler: ，此调用会提供session的identifier 引起你的app重启。你的app应该存储 实现处理（completion handler），用相同的identifier创建后台configuration object，再用此configuration object 创建session。新的session 自动重新关联到正在进行的后台活动。随后，当session 完成最后的下载任务，它发送给session delegate 一个 URLSessionDidFinishEventsForBackgroundURLSession: 消息。你的session delegate 然后调用已存储的实现处理（completion handler）

在iOS和OS X ,当用户重启app，你的app会立即使用相同的identier创建background configuration objects 作为任何session的显著的tasks，然后创建session为每一个configuration objects。这些新的session会自动重新关联到正在进行的后台活动。

如果在app暂停的时候任务完成可，代理的URLSession:downloadTask:didFinishDownloadingToURL: 会被调用，task和url会被关联到最新的下载文件里

类似的，如果有任何的task请求证书，NSURLSession会调用URLSession:task:didReceiveChallenge:completionHandler: method 或 URLSession:didReceiveChallenge:completionHandler:

在网络出错的时候上传和下载的后台任务会自动重试，通过URL下载系统。使用reachability APIs去决定设施重启失败的task是不必要的。

如何使用background transfers，见Simple Background Transfer这章.

##Life Cycle and Delegate Interaction
依据你想用session做什么，完全理解 session life cycle是有帮助的，包括一个session如何和他的Delegate交互，Delegate的调用顺序，当server返回一个重定向是发生了什么等等等~
完全的描述URL session 的life cycle，read Life Cycle of a URL Session

##NSCopying Behavior
Session and task objects 完全遵守 NSCopying 协议  

 * 当你的app 拷贝了session或task object，你会得到相同的object  
 * 当你的app 拷贝了configuration object，你会得到一个新的副本，你可以独立的修改他
 
##Creating and Configuring a Session
NSURLSession API 提供了广泛的配置选项  

* 私有存储以一种特别的单独seesion对于caches,cookies,credentials,protocols提供支持
* 证书，绑定在一个特别的请求或一组请求
* 通过URL上传文件和下载，它鼓励从元数据分离数据
* 每个host可以配置最大数量的连接
* 如果整个资源在一定时间内无法下载，每个资源的超时会被触发
* 有最小和最大化的TLS支持
* 自定义代理字典
* 可控制 cookie 策略
* 控制http管道行为

由于大多数设置已包含进单独的configuration object，你可以重用一般的setting。当你实例化一个session object，你需要指定以下规则：

* 一个 configuration object 在内部统治了session和tasks的行为
* 可选的，代理对象可以处理输入的数据和针对指定的session，task处理相关的事件。类如服务认证，决定是否一个资源加载请求转换为下载，等等

如果你不提供代理，NSURLSession 会使用系统提供的代理。用这种方式，你可以容易的使用  sendAsynchronousRequest:queue:completionHandler: 方法调用
	
	note：后台传输必须提供自定义delegate

在你初始化session object后,在不重新创建新的session情况下，你无法改变configuration或delegate  

Listing 1-2 shows examples of how to create normal, ephemeral, and background sessions.

	#if TARGET_OS_IPHONE

    self.completionHandlerDictionary = [NSMutableDictionary dictionaryWithCapacity:0];

	#endif

    /* Create some configuration objects. */ 

    NSURLSessionConfiguration *backgroundConfigObject = [NSURLSessionConfiguration backgroundSessionConfiguration: @"myBackgroundSessionIdentifier"];

    NSURLSessionConfiguration *defaultConfigObject = [NSURLSessionConfiguration defaultSessionConfiguration];

    NSURLSessionConfiguration *ephemeralConfigObject = [NSURLSessionConfiguration ephemeralSessionConfiguration];

    /* Configure caching behavior for the default session.

       Note that iOS requires the cache path to be a path relative

       to the ~/Library/Caches directory, but OS X expects an

       absolute path.

     */

	#if TARGET_OS_IPHONE

    NSString *cachePath = @"/MyCacheDirectory";

    NSArray *myPathList = NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES);

    NSString *myPath    = [myPathList  objectAtIndex:0];


    NSString *bundleIdentifier = [[NSBundle mainBundle] bundleIdentifier];


    NSString *fullCachePath = [[myPath stringByAppendingPathComponent:bundleIdentifier] stringByAppendingPathComponent:cachePath];

    NSLog(@"Cache path: %@\n", fullCachePath);

	#else

    NSString *cachePath = [NSTemporaryDirectory() stringByAppendingPathComponent:@"/nsurlsessiondemo.cache"];
    
    NSLog(@"Cache path: %@\n", cachePath);

	#endif

    NSURLCache *myCache = [[NSURLCache alloc] initWithMemoryCapacity: 16384 diskCapacity: 268435456 diskPath: cachePath];

    defaultConfigObject.URLCache = myCache;

    defaultConfigObject.requestCachePolicy = NSURLRequestUseProtocolCachePolicy;

    /* Create a session for each configurations. */

    self.defaultSession = [NSURLSession sessionWithConfiguration: defaultConfigObject delegate: self delegateQueue: [NSOperationQueue mainQueue]];

    self.backgroundSession = [NSURLSession sessionWithConfiguration: backgroundConfigObject delegate: self delegateQueue: [NSOperationQueue mainQueue]];

    self.ephemeralSession = [NSURLSession sessionWithConfiguration: ephemeralConfigObject delegate: self delegateQueue: [NSOperationQueue mainQueue]];

除了background configuration例外，你可以重用session configuration objects来创建额外的sessions。（你不能重用background session configurations，因为两个background session configurations 的行为共享相同的identifier 是未定义的）

你可以安全的修改configuration objects 在任意时刻。当你创建一个session，这个session执行一个深拷贝，所以修改置灰影响新的session，而不是已存在的session

##Fetching Resources Using System-Provided Delegates
最直接的方式就是用系统的函数，仅需要做一下两件事：  

 *  创建configuration object和session  
 *  在数据完全返回后处理回调内容


	    NSURLSession *delegateFreeSession = [NSURLSession sessionWithConfiguration: defaultConfigObject delegate: nil delegateQueue: [NSOperationQueue mainQueue]];
	    
	    [[delegateFreeSession dataTaskWithURL: [NSURL URLWithString: @"http://www.example.com/"]

    	                   completionHandler:^(NSData *data, NSURLResponse *response,

                                   NSError *error) {

                           NSLog(@"Got response %@ with error %@.\n", response, error);

                           NSLog(@"DATA:\n%@\nEND DATA\n",

                                 [[NSString alloc] initWithData: data

                                         encoding: NSUTF8StringEncoding]);

                       }] resume];



##Fetching Data Using a Custom Delegate
如果使用自定义代理，须执行至少一下两种函数

 * URLSession:dataTask:didReceiveData:  从你的请求任务内提供数据，一次一小块
 * URLSession:task:didCompleteWithError: 指出你的任务数据完全接受
 
 如果你的app需要在 URLSession:dataTask:didReceiveData: 返回后使用数据，你的代码需要以别的方式存储它
 
 Listing 1-5  Data task example
	
	NSURL *url = [NSURL URLWithString: @"http://www.example.com/"];
    NSURLSessionDataTask *dataTask = [self.defaultSession dataTaskWithURL: url];

    [dataTask resume];
    
##Downloading Files
在更高的级别，下载文件类似于获取data，你的app应该执行以下delegate 

 * URLSession:downloadTask:didFinishDownloadingToURL: 给你的提供一个临时文件位置存储已下载的内容
 
	    Important: 在函数返回之前，要么打开文件读取，要么移动到一个持久化的位置。当函数返回时，这个临时文件会被删除
	 
 * URLSession:downloadTask:didWriteData:totalBytesWritten:totalBytesExpectedToWrite: 提供下载进程信息
 * URLSession:downloadTask:didResumeAtOffset:expectedTotalBytes: 成功地恢复了一个先前失败的下载
 * URLSession:task:didCompleteWithError: 下载失败

如果你安排的是一个后台下载，当你的app不在运行时下载会继续。使用另外两个，当app重启是会开启新的下载

参考文档链接：  
[www.raywenderlich.com](https://www.raywenderlich.com/110458/nsurlsession-tutorial-getting-started)  
[developer.apple.com](https://developer.apple.com/library/prerelease/ios/documentation/Cocoa/Conceptual/URLLoadingSystem/Articles/UsingNSURLSession.html#//apple_ref/doc/uid/TP40013509-SW1)  
[objccn.io](http://objccn.io/issue-5-4/)  

