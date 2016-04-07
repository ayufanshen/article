---
layout: post
title: "FaceBook pop框架实战教程（二）"
description: ""
category: ios开发
tags: [ui]
author: 饭小团
---

##FaceBook pop框架实战教程（二）

本期之作品转场动效，前提是你已看过前面的一篇关于转场动效的文章。
本期内容打算在此基础上替换掉系统本身的动画UIView的api使用，转而使用pop的api。使一个控制器能从其他地方进入，并带有其他丰富的效果。

####[完整项目点这里](https://github.com/ayufanshen/PopAnimation)

####具体效果如下图所示

........![PopAnimation](../attachment/fanshen/PopAnimation2.gif)........

需要注意的是本篇的转场动效的代理没有遵循UINavigationController的UINavigationControllerDelegate协议，而是遵循的是UIViewControllerTransitioningDelegate。

原因是这里的效果是presenting一个ViewController，而不是原先的UINavigationController的导航跳转。


先把UIViewControllerTransitioningDelegate加进去。
接着填入以下代码

	-(void)viewDidLoad{

		//在前面的代码后补上
	    [self addPresentButton];

	}
	
	
	- (void)addPresentButton
	{
    	UIButton *presentButton = [UIButton buttonWithType:UIButtonTypeSystem];
	    presentButton.backgroundColor = [UIColor redColor];
    	[presentButton setTitle:@"Present Modal View Controller" forState:UIControlStateNormal];
	    [presentButton addTarget:self action:@selector(present:) forControlEvents:UIControlEventTouchUpInside];
		[self.view addSubview:presentButton];
    
    	[presentButton mas_makeConstraints:^(MASConstraintMaker *make) {
        	make.centerY.mas_equalTo(self.view).offset(240);
	        make.centerX.mas_equalTo(self.view);
    	    make.size.mas_equalTo(CGSizeMake(240, 40));
        
	    }];
	}

	- (void)present:(id)sender
	{
		ModalViewController *modalViewController = [ModalViewController new];
		//加入转场代理
		modalViewController.transitioningDelegate = self; 
		//定义转场类型是Custom
		modalViewController.modalPresentationStyle = UIModalPresentationCustom;
    
		[self presentViewController:modalViewController
        	               animated:YES
            	         completion:NULL];
	}

ModalViewController 就是一个viewController。里面放一个button用来dismiss。具体代码不上了。

下面要加入两个转场代理函数的实现，在presenting和dismiss时,这两个代理函数会返回一个动画对象.在转场过渡时会分别执行这两个动画效果

	#pragma mark - UIViewControllerTransitioningDelegate
	
	//presentViewController时会执行这个动效
	- (id<UIViewControllerAnimatedTransitioning>)animationControllerForPresentedController:	(UIViewController *)presented presentingController:(UIViewController *)presenting sourceController:(UIViewController *)source
	{
    	return [PresentingAnimator new];
	}

	
	- (id<UIViewControllerAnimatedTransitioning>)animationControllerForDismissedController:	(UIViewController *)dismissed
	{
    	return [DismissingAnimator new];
	}

你的PresentingAnimator和DismissingAnimator一定在报错，下面创造这两个对象，继承NSObject并加入遵循这个代理\<UIViewControllerAnimatedTransitioning>

先看PresentingAnimator
PresentingAnimator.h

	#import <Foundation/Foundation.h>
	#import <UIKit/UIKit.h>

	@interface PresentingAnimator : NSObject <UIViewControllerAnimatedTransitioning>

	@end
	
PresentingAnimator.m

	#import "PresentingAnimator.h"
	#import <POP/POP.h>

	@implementation PresentingAnimator

	- (NSTimeInterval)transitionDuration:(id <UIViewControllerContextTransitioning>)transitionContext
	{
    	return 0.5f;
	}

	- (void)animateTransition:(id <UIViewControllerContextTransitioning>)transitionContext
	{
		//获得来源view
    	UIView *fromView = [transitionContext viewControllerForKey:UITransitionContextFromViewControllerKey].view;
	    fromView.tintAdjustmentMode = UIViewTintAdjustmentModeDimmed;
    	fromView.userInteractionEnabled = NO;
    	
		//增加一个背景变暗的view
	    UIView *dimmingView = [[UIView alloc] initWithFrame:fromView.bounds];
    	dimmingView.backgroundColor = [UIColor grayColor];
	    dimmingView.layer.opacity = 0.0;
	    
	    
		//获得目标view
    	UIView *toView = [transitionContext viewControllerForKey:UITransitionContextToViewControllerKey].view;
	    toView.frame = CGRectMake(0,0,
                              CGRectGetWidth(transitionContext.containerView.bounds) - 104.f,
                              CGRectGetHeight(transitionContext.containerView.bounds) - 288.f);
    
    	toView.center = CGPointMake(transitionContext.containerView.center.x, -transitionContext.containerView.center.y);

		//把dimmingView和toView都加到当前transitionContex.containerView上
	    [transitionContext.containerView addSubview:dimmingView];
    	[transitionContext.containerView addSubview:toView];

    	POPSpringAnimation *positionAnimation = [POPSpringAnimation animationWithPropertyNamed:kPOPLayerPositionY];
	    positionAnimation.toValue = @(transitionContext.containerView.center.y);
    	positionAnimation.springBounciness = 10;
	    [positionAnimation setCompletionBlock:^(POPAnimation *anim, BOOL finished) {
    	    [transitionContext completeTransition:YES];//结束后要执行此函数告诉系统动画结束
	    }];

    	POPSpringAnimation *scaleAnimation = [POPSpringAnimation animationWithPropertyNamed:kPOPLayerScaleXY];
	    scaleAnimation.springBounciness = 20;
    	scaleAnimation.fromValue = [NSValue valueWithCGPoint:CGPointMake(1.2, 1.4)];

	    POPBasicAnimation *opacityAnimation = [POPBasicAnimation animationWithPropertyNamed:kPOPLayerOpacity];
    	opacityAnimation.toValue = @(0.2);

	    [toView.layer pop_addAnimation:positionAnimation forKey:@"positionAnimation"];
    	[toView.layer pop_addAnimation:scaleAnimation forKey:@"scaleAnimation"];
	    [dimmingView.layer pop_addAnimation:opacityAnimation forKey:@"opacityAnimation"];
	}

	@end

###### 注意：transitionContext.containerView 是个通明的view，你需要在这个上面添加子view，并让子view执行动画。

###### 在动画结束后Debug View Hierarchy。 可以很清楚的看见一个UITranstionView在灰色view后面

这样presenting的动画就可以运行了。
下面来看一下dismiss如何动效

	DismissingAnimator.h
	
	#import <Foundation/Foundation.h>
	#import <UIKit/UIKit.h>
	@interface DismissingAnimator : NSObject <UIViewControllerAnimatedTransitioning>

	@end


	DismissingAnimator.m
	#import "DismissingAnimator.h"
	#import <POP/POP.h>

	@implementation DismissingAnimator

	- (NSTimeInterval)transitionDuration:(id <UIViewControllerContextTransitioning>)transitionContext
	{
		return 0.5f;
	}

	- (void)animateTransition:(id <UIViewControllerContextTransitioning>)transitionContext
	{
    
    	UIViewController *fromVC = [transitionContext 		viewControllerForKey:UITransitionContextFromViewControllerKey];
    
    	UIViewController *toVC = [transitionContext 		viewControllerForKey:UITransitionContextToViewControllerKey];
    	toVC.view.tintAdjustmentMode = UIViewTintAdjustmentModeNormal;
    	toVC.view.userInteractionEnabled = YES;

    	//找见dimmingView
    	__block UIView *dimmingView;
    	[transitionContext.containerView.subviews enumerateObjectsUsingBlock:^(UIView *view, NSUInteger idx, BOOL *stop) {
       	 	if (view.layer.opacity < 1.f) {
          	  dimmingView = view;
           		 *stop = YES;
       		 }
    	}];
    
    	POPBasicAnimation *opacityAnimation = [POPBasicAnimation animationWithPropertyNamed:kPOPLayerOpacity];
    	opacityAnimation.toValue = @(0.0);

    	POPBasicAnimation *offscreenAnimation = [POPBasicAnimation animationWithPropertyNamed:kPOPLayerPositionY];
    	offscreenAnimation.toValue = @(-fromVC.view.layer.position.y);
    	[offscreenAnimation setCompletionBlock:^(POPAnimation *anim, BOOL finished) {
        	[transitionContext completeTransition:YES];
		}];
    	[fromVC.view.layer pop_addAnimation:offscreenAnimation forKey:@"offscreenAnimation"];
    	[dimmingView.layer pop_addAnimation:opacityAnimation forKey:@"opacityAnimation"];
	}

	@end

就是这样了，在ViewController被dismiss时，执行动画。结束后dimmingView和modelViewController的view也全部消除了。
run一下，看看效果如何~ 


