#iOS 保持界面流畅的技巧




[iOS 保持界面流畅的技巧](http://blog.ibireme.com/2015/11/12/smooth_user_interfaces_for_ios/)，这篇文章很干货，下面内容是做笔记。



1. iOS卡顿的原因。
	![](ios_screen_display.png)
	CPU计算好需要显示的内容，提交给GPU，GPU渲染好后，提交给帧缓冲区。  
	![](ios_screen_scan.png)
	![](ios_vsync_off.jpg) 
	为了解决帧缓冲区的读写冲突问题，设计了两个缓冲区，一个用于读，一个用于写，并且用显示器的V-Sync信号，作为两个缓冲区的橘色切换。 
 	![](ios_frame_drop.png)
 	如果V-Sync信号来了，新的渲染结果，还没有写入到缓冲区，就会出现掉帧的情况。

2. 列出可以优化的关键点
    * CPU层面：优化计算量
      * 对象创建  
      使用轻量对象
      * 对象调整，消息传递  
      尽量减少调整
      * 对象销毁  
      放到后台线程。  
      上面说的三点，因为UI和CA层对象不是线程安全的，对他们的操纵和访问，不能放到后台线程。
      * 布局计算
      * autolayout
      * 文本高度、宽度计算   
      上面说的几点，可以通过缓存计算结果来优化，避免重复计算
      * 文本在coretext层渲染，高度和宽度又会计算一遍  
      用 TextKit 或最底层的 CoreText 对文本异步绘制。
      * 图片解码  
      这一步是发生在主线程的，并且不可避免。如果想要绕开这个机制，常见的做法是在后台线程先把图片绘制到 CGBitmapContext 中，然后从 Bitmap 直接创建图片。目前常见的网络图片库都自带这个功能
      * CoreGraphic层绘制  
      放到后台绘制好后，然后异步交给主线程
	* GPU层面：优化接受纹理、顶点描述、变换、混合、渲染、输出到屏幕上的过程
      * 业务上避免过多图片的设计
      * 在cpu那里，可以将多个图层合成为一个图层
      * 图片像素和屏幕像素要匹配，图片不要超过gpu的最大承受力
      * 不透明的视图，表名apaue，避免无用的alpha通道合成
      * 避免使用圆角、阴影、遮罩等属性，尝试设置shouldRasterize将这部分工作转到cpu，或者用其他手段，达到相同的视觉效果。
      

3. ASDK架构
  ASDK 认为，阻塞主线程的任务，主要分为
	* 文本高度计算、视图布局计算
 	* 文本渲染、图片解码、图形绘制
 	* UIKit Objects的创建、调整、销毁
  
	文本和布局的计算、渲染、解码、绘制都可以通过各种方式异步执行，但 UIKit 和 Core Animation 相关操作必需在主线程进行。**ASDK 的目标，就是尽量把这些任务从主线程挪走（利用多核），而挪不走的，就尽量优化性能（利用合成，减少UIView和CALayer的创建）**。   
  ASDK是重新实现封装了一些控件，这些控件和UIKit的控件一一对应，但内部却是基于CoreAnimation层和UIKit层重新实现的。   
  UIKit和CoreAnimation关系：
  ![](asdk_layer_backed_view.png)·chr
  需要响应动作时，ASDK封装UIKit，用来实现
  ![](asdk_view_backed_node.png)
  不需要响应动作时，ASDK封装CoreAnimation
  ![](asdk_layer_backed_node.png)

4. ASDK实现

	1. ASDispalyNode是线程安全的，他重新实现了大部分UI控件，实现了上述的优化关键点。比如：

		1. 当多个视图叠加时，不需要响应事件的视图，会合成为图片，减少了UIView和CALayer的创建，从而减轻CPU和GPU负担。  
		注意设置backed
		2. 异步并发，将可以放到后台线程做的事情，移除主线程。
比如布局计算、排版、解码等，封装成较小的任务，并利用GCD异步并发执行

	2. runloop 任务分发  
	Core Animation 在 RunLoop 中注册了一个 Observer，监听了 BeforeWaiting 和 Exit 事件。这个 Observer 的优先级是 2000000，低于常见的其他 Observer。当一个触摸事件到来时，RunLoop 被唤醒，App 中的代码会执行一些操作，比如创建和调整视图层级、设置 UIView 的 frame、修改 CALayer 的透明度、为视图添加一个动画；这些操作最终都会被 CALayer 捕获，并通过 CATransaction 提交到一个中间状态去（CATransaction 的文档略有提到这些内容，但并不完整）。当上面所有操作结束后，RunLoop 即将进入休眠（或者退出）时，关注该事件的 Observer 都会得到通知。这时 CA 注册的那个 Observer 就会在回调中，把所有的中间状态合并提交到 GPU 去显示；如果此处有动画，CA 会通过 DisplayLink 等机制多次触发相关流程。  
	ASDK 在此处模拟了 Core Animation 的这个机制：所有针对 ASNode 的修改和提交，总有些任务是必需放入主线程执行的。当出现这种任务时，ASNode 会把任务用 ASAsyncTransaction(Group) 封装并提交到一个全局的容器去。ASDK 也在 RunLoop 中注册了一个 Observer，监视的事件和 CA 一样，但优先级比 CA 要低。当 RunLoop 进入休眠前、CA 处理完事件后，ASDK 就会执行该 loop 内提交的所有任务。具体代码见这个文件：ASAsyncTransactionGroup。  
**通过这种机制，ASDK 可以在合适的机会把异步、并发的操作同步到主线程去，并且能获得不错的性能。**

5. 做了一个Demo，从CPU层面进行优化。



