##			     Facebook 如何打造 iOS App 高性能滚屏
   为用户传递一个最好的体验是我们的目标之一。部分目标就是要确保新闻订阅的平滑过渡。但是在一个带有多个多种变量复杂的scroll view里，这里目前没有一个好办法用来确定在哪里的帧数会下降。我们开发出了一个识别策略，他在实践中工作的非常好，并且帮助我们维持了一个高性能scroll。
   
####在设备上测量scroll的性能
Apple's Instruments app 允许你测量你的app帧率，但是在模拟交互行为上却变得非常困难。用其他方式可以在真机上测量其scroll的性能。

我们使用 Apple's CADisplayLink API 用来测量，每一次的帧被渲染，我们测量时间的消耗。如果花费的时间超过16.6ms，其帧数下降，滑动迟缓。

	[CADisplayLink displayLinkWithTarget:self selector:@selector(_update)];
	[_displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
	
#####发现并修复它
和视频游戏不一样，facebook app并不是非常依赖GPU。他大多数是用来展示图片和文字，因此多数的帧下降是来自于CPU。
为了维持一个健壮的cpu，我们必须确保在News Feed的所有渲染操作的执行时间小于16.6ms。实际上，渲染一个页面是由多个步骤组成的，而且一个典型的app在帧数下降之前仅有8至10毫秒的主线程执行时间，并不是完整的16.6ms。

知道主线程在哪里花费CPU对于获得最好的滑动体验是非常有帮助的，Time Profiler也是可以计算主线程时间，但对于精确的情况却困难很多。

我们打算用信号来进行性能测量，虽然会有噪音，但是他可以允许你在沙盒环境下获得性能数据。传统的ios工具和DTrace是不可能的。

####信号与真机分析
为了了解线程做了什么，我们通过信号暂停他，并发送一个有回调函数的信号

	static void _callstack_signal_handler(int signr, siginfo_t *info, void *secret) {
    callstack_size = backtrace(callstacks, 128);
	}

	struct sigaction sa;
	sigfillset(&sa.sa_mask);
	sa.sa_flags = SA_SIGINFO;
	sa.sa_sigaction = _callstack_signal_handler;
	sigaction(SIGPROF, &sa, NULL);
	
这个操作有非常好的信号安全限制，对于内存的分配就不是了。所以我们做的唯一的事就是获取当前回调。

####发送信号

![Alt text](1.png)

我们需要一个非主线程的线程来发送信号，GCD不错，但不能没10ms执行一次，所以我们用NSThread。
我们要给与NSThread更高的权限来执行操作，以防主线程过于忙碌影响了我们的发信号的线程。

	_trackerThread = [[NSThread alloc] initWithTarget:[self class] 	selector:@selector(_trackerLoop) object:nil];    
	_trackerThread.threadPriority = 1.0;
	[_trackerThread start];
这就是一个普通的性能测试方案，实际上这个测试行为依然隐式的影响了app的性能。在iPhone4s上，他大约花费了1ms。相对于16ms的时间，1ms实在很小。同样的，发送信号的行为悬挂了主线程，在线程间产生了更复杂的环境切换，并且在减速了整体app。

选择一个合适的抽样方案是非常重要的，只有在绝对必要的时候再测量他。在我们这里，当我们开始抽样时，我们会制作很多优化方案。最直接的例子就是，当用户滑动scroll时才发送信号。其他情况就是我们只在内部测试，而不放到线上的app上。

####生成日志并符号化

当backtrace生成数据的时候，我们在服务器收集到他。backtrace是无法读的。需要symbolicate。Apple atos API,Google Breakpad, andFacebook's atosl 均可以符号化他。符号化数据后使用data-visualization工具就可以确定是那些地方需要我们优化scroll 性能。

以下是一个两个版本的app CPU性能消耗的例子：

![Alt text](2.png)

####Try it out

This strategy has allowed us to detect a very large number of regressions before they hit production. We put a sample of this implementation onGitHub. We hope you will find it useful in projects of your own! 
