#AsyncDisplayKit  

##Core Concepts   

###Intelligent Preloading  
当一个node有能力被渲染，异步测量，并发会使它变得更加强大，对于ASDK的重要一面就是智能预加载。  

如前文指出的，脱离任何一个node容器使用node都不会带来多少好处。这是由于所有node都有一个关于他们当前接口状态的慨念。  

interfaceState属性是被一个ASRangeController持续更新的，ASRangeController
内部创建和维护了所有容器。  

一个node在容器外使用时不会被任何range controller持续更新的。当node被渲染时，有时会导致一闪，这是在node意识到他们已经onscreen，而且没有任何警告之后。  

###Interface State Ranges  
当node被添加到一个scrolling或paging接口时,他们典型的处于一个following ranges。这意味着当scrolling view 正在滑动时，他们的interface states会被持续更新，当node通过屏幕。  

Interface State 	  | Description   
Fetch Data: 远离visible的距离时，这里的内容会从外部收集，从api或硬盘。  
Display	: 这里，display的任务是文字光栅处理和图片解码发生的地方。  
Visible	: node被显示至少一像素。  

###ASRangeTuningParameters  
这些range的每一个size都会被全屏测量，虽然在很多情况下默认size会工作的很好，他们可以容易调整通过设置tuning parameters对于range type，在你scrolling node时。  
图：

在以上的scrolling collection，用户在向下滑动。正如你所看，下面的范围size比上面的大一点。如果用户改变滑动方向，头尾两边会动态交换，以便优化内存。这样你就值考虑头尾size大小，而不用担心用户改变滑动方向时的响应。

聪明的预加载同样能工作在多个场景~  

###Interface State Callbacks  
Visible Range

- (void)visibilityDidChange:(BOOL)isVisible;
Display Range

- (void)displayStateDidChange:(BOOL)inDisplayState;
Fetch Data Range

- (void)loadStateDidChange:(BOOL)inLoadState;

Just remember to call super ok? 😉  

##Node Containers  
###Node Containers  
强烈建议你使用node container ~

	* ASDK Node Container 	 * UIKit Equivalent

	ASCollectionNode 	    | in place of UIKit's UICollectionView  
	ASPagerNode             | in place of UIKit's UIPageViewController  
	ASTableNode             | in place of UIKit's UITableView  
	ASViewController        | in place of UIKit's UIViewController  
	ASNavigationController  | in place of UIKit's UINavigationController.
	ASTabBarController      | in place of UIKit's UITabBarController.
	                        | Implements the ASVisibility protocol.

###What do I Gain by Using a Node Container?
一个node container 自动处理node的intelligent preloading，这意味着所有node的布局，数据抓取，解码和渲染都会被异步完成。  

##Node Subclasses  
使用node而不是UIKit的组件的关键好处是在主线程外，所有node执行了layout和显示。所以主线程可以立刻响应用户交互事件。  

	ASDK Node 	         |  UIKit Equivalent
	------------------------------------------------
	ASDisplayNode 	      | in place of UIKit's UIView
                          | The root AsyncDisplayKit node, from which all other nodes inherit.
                    
    ASCellNode            | in place of UIKit's UITableViewCell & UICollectionViewCell 
                          | ASCellNodes are used in ASTableNode, ASCollectionNode and ASPagerNode.
                    
    ASScrollNode          | in place of UIKit's UIScrollView
                          | This node is useful for creating a customized scrollable region that contains other nodes.
                          
    ASEditableTextNode    | in place of UIKit's UITextView
    ASTextNode            | in place of UIKit's UILabel  
    
	ASImageNode           | in place of UIKit's UIImage
	ASNetworkImageNode
	ASMultiplexImageNode 	
	
    ASVideoNode			   | in place of UIKit's AVPlayerLayer
    ASVideoPlayerNode      | in place of UIKit's UIMoviePlayer


    ASControlNode 	       | in place of UIKit's UIControl
	ASButtonNode           | in place of UIKit's UIButton
	ASMapNode 	           | in place of UIKit's MKMapView


图：  
蓝色的node是UIKit的封装，如：ASScrollNode wraps a UIScrollView, and ASCollectionNode wraps a UICollectionView.   

##Subclassing  
无论你创建ASViewController或ASDisplayNode，创建子类是最重要的区别。这很明显，但由于一些微妙的不同，需要保持大脑清醒~  

###ASDisplayNode  
子类化nodes就类似于子类化UIView，这里有一些指导用来确定既能完全利用其framework潜能，还能够让node的行为是你期望的~  

####-init  
当使用nodeBlocks时,这个方法会被后台线程调用。然而，因为没有其他函数能在init完成前跑起来，所以不需要lock它。  
最重要的是，你的init函数可以在任何队列调用。最显著的是，你永远不能初始化任何UIKit 对象，点击view或node的layer，不能添加任何手势。想做这些事，在-didLoad里实现。  

####-didLoad  
这个函数类似于-viewDidLoad，这里backing view被加载。它保证在主线程里被调用，在合适的地方做UIKit的事情（手势识别，触摸view/layer，在初始化UIKit对象）  

####-layoutSpecThatFits:  
这个函数在后台线程定义了layout和处理繁重的计算。这个函数就是你建立布局空间对象以便计算node的size，同样对于subnodes的size和position。这里就是你放置多数布局代码的地方。
由于他是在后台线程运行，你不必设置任何node.view 或 node.layer 位置。除非你知道你在做什么，不要创建任何node在这里。此外，也没必要在这个函数开始时call super，除非其他函数override。  

####-layout  
稍后。。。 

####ASViewController 
ASViewController是一个合格的UIViewController 子类，有个特别的功能管理node。由于是UIViewController的子类，所有函数都在主线程调用（你应该总是在主线程上创建ASViewController）

####-init  
这个函数只调用一次，在ASViewController的生命初期。当UIViewController初始化完毕后，最好的实践就是永远不要在这里访问self.view 或 self.node.view，它会强制view过早的创建。代替的是在 -viewDidLoad里访问。  


ASViewController 的初始化器是initWithNode。一个典型的初始化器如下所示。注意ASViewController的node在调super前就被创建了。一个ASViewController管理一个node就像UIViewController管理一个view，只是初始化有些不同。  

	- (instancetype)init
	{
	  	_pagerNode = [[ASPagerNode alloc] init];
		self = [super initWithNode:_pagerNode];

		// setup any instance variables or properties here
	  	if (self) {
			pagerNode.dataSource = self;
			pagerNode.delegate = self;
	  	}	
	  	return self;
	}

####-loadView 
不要用它

####-viewDidLoad 
只在ASViewController的生命初期call一次。这里是访问node的最早的时间，这块是最好的地方来放置只跑一次的代码和需要访问的view/layer，比如添加手势。  

布局代码不要放这里，因为当有变化的时候他不会再次调用。这对UIViewController也是一样的。即使你当前不需要有几何变化，也不要把layout代码放这里。 

####-viewWillLayoutSubviews  
这个函数和node的layout完全同时调用，会在ASViewController生命周期内多次调用。无论何时ASViewController的node的bound改变时候。（including rotation, split screen, keyboard presentation）同样的当个层级改变的时候（children being added, removed, or changed in size)。
为了保持一致性，最好的方案就是把所有layout的代码都放进次函数，因为这里不会频繁调用，即使代码不是直接依赖这里的尺寸。

####-viewWillAppear: / -viewDidDisappear:
这个函数是在ASViewController的node显示在屏幕显示之前和显示之后调用。这个函数提供一个很好的机会用于开始或停止动画，对于presentation 或 dismissal 你的控制器。

##Node Containers
ASViewController是个UIViewController的子类，增加了几个属于ASDisplayNode层级的功能。

ASViewController可以用于代替任何UIViewController，包括UINavigationController, UITabBarController and UISpitViewController 或者modal view controller 

使用ASViewController最主要的好处就是省内存。一个ASViewController可以进行离屏渲染，将会减少fetch data 并且显示任何范围内的node。
