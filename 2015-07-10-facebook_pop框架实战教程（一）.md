---
layout: post
title: "FaceBook pop框架实战教程（一）"
description: ""
category: ios开发
tags: [ui]
author: 饭小团
---

##FaceBook pop框架实战教程（一）

#####本文要使用Facebook的pop框架制作两个控件。有弹性的button和弹出式Lable

####[完整项目点这里](https://github.com/ayufanshen/PopAnimation)

####实际效果如下图所示

........![PopAnimation](../attachment/fanshen/PopAnimation.gif)........

首先新建一个项目，需要导入pop框架和Masonry框架。下载地址点[这里pop](https://github.com/facebook/pop)和这里[这里Masonry](https://github.com/SnapKit/Masonry)

导入这两个框架后就可以把他们加入FirstViewController里（把原来的ViewController改名为FirstViewController，因为后面转场动画时还要用它）

开始先做一个弹性的button吧。
继承一个UIButton，PopButton

PopButton.h里是这样

	@interface PopButton : UIButton

		+ (instancetype)button;

	@end

PopButton.m是这样

	@implementation PopButton

	+ (instancetype)button
	{
    	return [self buttonWithType:UIButtonTypeCustom];
	}

	- (id)initWithFrame:(CGRect)frame
	{
    	self = [super initWithFrame:frame];
	    if (self) {
       		[self setup];
	    }
    	return self;
	}

	- (void)setup
	{
		self.backgroundColor = self.tintColor;
    	self.layer.cornerRadius = 4.f;
	    self.titleLabel.font = [UIFont fontWithName:@"Avenir-Medium"
                                           size:22];
    
    	[self addTarget:self action:@selector(scaleToSmall)
	   forControlEvents:UIControlEventTouchDown| UIControlEventTouchDragEnter];
    
    	[self addTarget:self action:@selector(scaleAnimation)
	   forControlEvents:UIControlEventTouchUpInside];
    
    	[self addTarget:self action:@selector(scaleToDefault)
	   forControlEvents:UIControlEventTouchDragExit];
	}
	
这里面没什么好说的，button在按下去的时候同时又执行了一次自己类中的函数。

下面使用了两个pop的类：

1.POPBasicAnimation 用于执行普通动画效果，平移，翻转。

2.POPSpringAnimation 是重点。很神奇~ 弹出，弹入动画主要靠他。

	- (void)scaleToSmall
	{

		//kPOPLayerScaleXY 告诉POPBasicAnimation 在X轴和Y轴放大
		//toValue 是放大倍数
		//最后加入动画
    	POPBasicAnimation *scaleAnimation = [POPBasicAnimation animationWithPropertyNamed:kPOPLayerScaleXY]; 
	    scaleAnimation.toValue = [NSValue valueWithCGSize:CGSizeMake(0.85f, 0.85f)];
    	[self.layer pop_addAnimation:scaleAnimation forKey:@"layerScaleSmallAnimation"];

	}
		
	- (void)scaleAnimation
	{
		//kPOPLayerScaleXY 告诉POPSpringAnimation 在X轴和Y轴反弹
		//velocity 是反弹速率倍数
		//toValue  是反弹回的大小倍数
		//springBounciness 是跳跃效果的弹出范围
		//最后加入动画
    	POPSpringAnimation *scaleAnimation = [POPSpringAnimation animationWithPropertyNamed:kPOPLayerScaleXY];
	    scaleAnimation.velocity = [NSValue valueWithCGSize:CGSizeMake(3.f, 3.f)];
    	scaleAnimation.toValue = [NSValue valueWithCGSize:CGSizeMake(1.f, 1.f)];
	    scaleAnimation.springBounciness = 18.0f;
    	[self.layer pop_addAnimation:scaleAnimation forKey:@"layerScaleSpringAnimation"];
	}

	//这个函数是用来复原		
	- (void)scaleToDefault
	{
    	POPBasicAnimation *scaleAnimation = [POPBasicAnimation animationWithPropertyNamed:kPOPLayerScaleXY];
	    scaleAnimation.toValue = [NSValue valueWithCGSize:CGSizeMake(1.f, 1.f)];
    	[self.layer pop_addAnimation:scaleAnimation forKey:@"layerScaleDefaultAnimation"];
	    NSLog(@"Default");
	}


好了，现在回到FirstViewController.m可以把这个button加进去了

	@interface FirstViewController (){
    
	}

	@property (nonatomic) PopButton *button;
	@property (nonatomic) UILabel *errorLabel;

	@end

	@implementation FirstViewController

	- (void)viewDidLoad {
    	[super viewDidLoad];
	    self.view.backgroundColor = [UIColor whiteColor];
	    
	    self.button = [PopButton button];
    	self.button.backgroundColor = [UIColor blueColor];
	    [self.button setTitle:@"表动我" forState:UIControlStateNormal];
    	[self.button addTarget:self action:@selector(touchUpInside) forControlEvents:UIControlEventTouchUpInside];
	    [self.view addSubview:self.button];
    
    	[self.button mas_makeConstraints:^(MASConstraintMaker *make) {
        	make.center.mas_equalTo(self.view);
	        make.size.mas_equalTo(CGSizeMake(100, 40));
    	}];

	}

记得把<pop/POP.h> <Masonry.h> PopButton.h三个东西导进去。

如果目前一切顺利的话，我们再在button后面加个可以弹出的label。

viewDidLoad 最下面加入这些

	- (void)viewDidLoad {
		。。。。。。。。。
		。。。。。。。。。
		
		[self addErrorLabel];
	}
	
	-(void)addErrorLabel{
    
    	self.errorLabel = [UILabel new];
	    self.errorLabel.font = [UIFont fontWithName:@"Avenir-Light" size:14];
    	self.errorLabel.textColor = [UIColor redColor];
	    self.errorLabel.text = @"乱点什么 ？别动! 烦着呢~";
    	self.errorLabel.textAlignment = NSTextAlignmentCenter;
    	//藏在buttonh后面
	    [self.view insertSubview:self.errorLabel belowSubview:self.button];
    
    	[self.errorLabel mas_makeConstraints:^(MASConstraintMaker *make) {
        	make.center.mas_equalTo(self.button);
	    }];
    	//缩小它，0.1倍
    	self.errorLabel.layer.transform = CATransform3DMakeScale(0.1f, 0.1f, 1.f);
	}

touchUpInside是button要执行的点击函数

	-(void)touchUpInside{
    
	//    [self hideLabel];
    
    	dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(0.5f * NSEC_PER_SEC)), 	dispatch_get_main_queue(), ^{
    	    [self showLabel];
	    });
	}

[self hideLabel] 先注掉，随后再写hide函数

	- (void)showLabel
	{
		
		// 使用POPSpringAnimation 弹性类放大 errorLabel
    	POPSpringAnimation *layerScaleAnimation = [POPSpringAnimation animationWithPropertyNamed:kPOPLayerScaleXY];
    	layerScaleAnimation.springBounciness = 18;
    	layerScaleAnimation.toValue = [NSValue valueWithCGSize:CGSizeMake(1.f, 1.f)];
    	[self.errorLabel.layer pop_addAnimation:layerScaleAnimation forKey:@"labelScaleAnimation"];
    
    	// 使用POPSpringAnimation 上移 errorLabel 注意这个kPOPLayerPositionY值是告诉类是针对Y轴移动
		POPSpringAnimation *layerPositionAnimation = [POPSpringAnimation animationWithPropertyNamed:kPOPLayerPositionY];
	    layerPositionAnimation.toValue = @(self.button.layer.position.y -50);
    	layerPositionAnimation.springBounciness = 12;
	    [self.errorLabel.layer pop_addAnimation:layerPositionAnimation forKey:@"layerPositionAnimation"];
    
	}
	
简单不？ 弹出弹入就是这个POPSpringAnimation类来回折腾
打开[self hideLabel]，加入以下函数

	- (void)hideLabel
	{	
		//搜小回0.1倍状态
    	POPBasicAnimation *layerScaleAnimation = [POPBasicAnimation animationWithPropertyNamed:kPOPLayerScaleXY];
	    layerScaleAnimation.toValue = [NSValue valueWithCGSize:CGSizeMake(0.1f, 0.1f)];
    	[self.errorLabel.layer pop_addAnimation:layerScaleAnimation forKey:@"layerScaleAnimation"];
    	
    	//下降回原始位置
	    POPBasicAnimation *layerPositionAnimation = [POPBasicAnimation animationWithPropertyNamed:kPOPLayerPositionY];
    	layerPositionAnimation.toValue = @(self.button.layer.position.y);
	    [self.errorLabel.layer pop_addAnimation:layerPositionAnimation forKey:@"layerPositionAnimation"];
	}

好了，就这样吧。run以下看看是不是想要的效果~