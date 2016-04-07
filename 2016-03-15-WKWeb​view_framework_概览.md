
---
layout: post
title: "WKWeb​View Framework 概览"
description: ""
category: ios
tags: net
author: 范珅
--- 


##WKWeb​View Framework 概览

iOS 与 web 之间的关系非常复杂，这种复杂关系甚至可以追溯到几十年前系统建立初期。

其实现在很难说清第一代 iPhone 横空出世是一件多么困难的事情。我们现今司空见惯的触摸屏在当时只是诸多方案中的一种。最早期的产品原型是物理键盘、触摸屏、触控笔的结合，屏幕尺寸才是 5" x 7"。甚至当时 iPod 的轮子都是一个严肃的备选方案。

但最最重要的决定或许都是由软件而非硬件决定的。

iPhone 应该如何运行软件呢？像 OS X 上的应用程序，或者 web 页面，以及 Safari 都应该如何运行起来呢？仿照 OS X 去构建 iPhone OS 的方法已经广泛地被大家熟知了，这种方法到今天也留下了不少争议。

Web 一直是 iOS 系统上的二级公民(讽刺的是，其实现今移动网页响应式设计的出现大多是 iPhone 推动的)。UIWebView 笨重难用，还有内存泄漏，和 Nirtro JavaScript 引擎谈笑风生的 Safari 不知道要比它高到哪里去了。

然而，这所有的一切都会因为 WKWebView 和 WebKit 框架其他部分的出现而发生改变。


WKWebView 是现代 WebKit API 在 iOS 8 和 OS X Yosemite 应用中的核心部分。它代替了 UIKit 中的 UIWebView 和 AppKit 中的 WebView，提供了统一的跨双平台 API。

自诩拥有 60fps 滚动刷新率、内置手势、高效的 app 和 web 信息交换通道、和 Safari 相同的 JavaScript 引擎，WKWebView 毫无疑问地成为了 WWDC 2014 上的最亮点。

UIWebView & UIWebViewDelegate 这个两个东西是如何在 WKWebKit 中被重构成 14 个类 3 个协议的呢。虽然这次的变化确实带来了不少的新功能，但请一定不要因此感到恐慌！

###WKWebKit Framework
####Classes
 
   * WKBackForwardList: 之前访问过的 web 页面的列表，可以通过后退和前进动作来访问到。
      *  WKBackForwardListItem: webview 中后退列表里的某一个网页。
   * WKFrameInfo: 包含一个网页的布局信息。
   * WKNavigation: 包含一个网页的加载进度信息。
      * WKNavigationAction: 包含可能让网页导航变化的信息，用于判断是否做出导航变化。
	  * WKNavigationResponse: 包含可能让网页导航变化的返回内容信息，用于判断是否做出导航变化。
   * WKPreferences: 概括一个 webview 的偏好设置。
   * WKProcessPool: 表示一个 web 内容加载池。
   * WKUserContentController: 提供使用 JavaScript post 信息和注射 script 的方法。
      * WKScriptMessage: 包含网页发出的信息。
      * WKUserScript: 表示可以被网页接受的用户脚本。 > - WKWebViewConfiguration: 初始化 webview 的设置。
   * WKWindowFeatures: 指定加载新网页时的窗口属性。


###Protocols

   * WKNavigationDelegate: 提供了追踪主窗口网页加载过程和判断主窗口和子窗口是否进行页面加载新页面的相关方法。
   * WKScriptMessageHandler: 提供从网页中收消息的回调方法。
   * WKUIDelegate: 提供用原生控件显示网页的方法回调。



###一、WKWebView新特性

   * 在性能、稳定性、功能方面有很大提升（最直观的体现就是加载网页是占用的内存，模拟器加载百度与开源中国网站时，WKWebView占用23M，而UIWebView占用85M）；
   * 允许JavaScript的Nitro库加载并使用（UIWebView中限制）；
   * 支持了更多的HTML5特性；
   * 高达60fps的滚动刷新率以及内置手势；
   * 将UIWebViewDelegate与UIWebView重构成了14类与3个协议（查看苹果官方文档）；
   
###二，例子🌰

	- (void)viewDidLoad {
    	[super viewDidLoad];
    
	    _webView = [[WKWebView alloc] initWithFrame:self.view.bounds configuration:config];
	    _webView.UIDelegate = self;
    	_webView.navigationDelegate = self;
	    [_webView loadRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:@"https://www.apple.com/cn/"]]];
    	[self.view addSubview:_webView];
    
    
	    progressView = [[UIProgressView alloc] initWithProgressViewStyle:UIProgressViewStyleDefault];
    	progressView.frame = CGRectMake(0, 65, self.view.bounds.size.width, 10);
	    [self.view addSubview:progressView];
	}

	#pragma mark - WKNavigationDelegate

	//准备加载页面
	- (void)webView:(WKWebView *)webView didStartProvisionalNavigation:(WKNavigation *)navigation{
    
    	self.title = @"准备加载页面";
    	progressView.hidden = NO;
    	[progressView setProgress:webView.estimatedProgress animated:YES];
	}

	//已开始加载页面，可以在这一步向view中添加一个过渡动画
	- (void)webView:(WKWebView *)webView didCommitNavigation:(WKNavigation *)navigation{

    	self.title = @"正在加载页面";
    	[progressView setProgress:webView.estimatedProgress animated:YES];
	}

	//页面已全部加载，可以在这一步把过渡动画去掉
	- (void)webView:(WKWebView *)webView didFinishNavigation:(WKNavigation *)navigation{
    
    	self.title = webView.title;
    	[progressView setProgress:webView.estimatedProgress animated:YES];
    	if (progressView.progress == 1) {
      	  [progressView setProgress:0 animated:YES];
      	  progressView.hidden = YES;
    	}
    
	}
	
###JavaScript ↔︎ OC/Swift 对话机制

相对于 UIWebView 最大的提升就是数据在可以 app 和 web 内容之间传递。

使用用户脚本来注入 JavaScript

###WKWebView加载JS
	
	// 图片缩放的js代码
		NSString *js = @"var count = document.images.length;for (var i = 0; i < count; i++){var image = document.images[i];image.style.width=600;}";
    
    // 根据JS字符串初始化WKUserScript对象
	    WKUserScript *script = [[WKUserScript alloc] initWithSource:js injectionTime:WKUserScriptInjectionTimeAtDocumentEnd forMainFrameOnly:NO];
    
    // 根据生成的WKUserScript对象，初始化WKWebViewConfiguration
    	WKWebViewConfiguration *config = [[WKWebViewConfiguration alloc] init];
    	[config.userContentController addUserScript:script];
    
    	_webView = [[WKWebView alloc] initWithFrame:self.view.bounds configuration:config];
    	[_webView loadHTMLString:@"<head></head><img src='http://pica.nipic.com/2008-05-31/2008531161451834_2.jpg' />" baseURL:nil];
    	[self.view addSubview:_webView];
    	

###结语
WKUserScript 对象可以以 JavaScript 源码形式初始化，初始化时还可以传入是在加载之前还是结束时注入，以及脚本影响的是这个布局还是仅主要布局。于是用户脚本被加入到 WKUserContentController 中，并且以 WKWebViewConfiguration 属性传入到 WKWebView 的初始化过程中。

如果你的 app 只是对网页内容做了很简单的一层包装，那么 WKWebView 可以彻底改变这种状况。你对于性能和兼容性的所有愿望都将变为现实。所有你想要的东西都在这了。

如果你是一个原生纯粹主义者，你可能会被 iOS 8 新带来强大和扩展性功能吓到。有一个秘密就是，例如 Messages 这种原生应用都应用了 WebKit 来渲染复杂的页面元素。你可能尚且没有意识到，但事实是，webview 在移动开发最佳实践中应该得到一席之地。
    	