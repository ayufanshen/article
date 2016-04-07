---
layout: post
title: "控制器间的转场效果的制作"
description: ""
category: ios开发
tags: [ui]
author: 饭小团
--- 


##控制器间的转场效果的制作
 ---
   转场动画的效果其实还是比较容易的，需要遵循UINavigationController的UINavigationControllerDelegate协议，并且在另一个对象中实现他即可。 
   
原教程地址是[这里](http://dativestudios.com/blog/2013/09/29/interactive-transitions/)，作者使用storyboard结合做的。我这里使用代码编写，方便各位读懂并搞清楚其效果。

最终的效果是这个样子的，第一个控制器点击图片后会平滑的过渡到第二个控制器。中间不会有明显的push效果。

![](../attachment/fanshen/transtion.png)

打开这个实例项目重点看一下FirstViewController.m文件。

	pragma mark UINavigationControllerDelegate methods
	- (id<UIViewControllerAnimatedTransitioning>)navigationController:(UINavigationController *)navigationController  animationControllerForOperation:(UINavigationControllerOperation)operation
                                     fromViewController:(UIViewController *)fromVC
                                       toViewController:(UIViewController *)toVC {
    
    if (fromVC == self && [toVC isKindOfClass:[SecondViewController class]]) {
        
        return [[TransitionFromFirstToSecond alloc] init];
        
    }else {
     	     return nil;
		}
	}

* 这个代理是必须实现的，他会在你push的时候调用这个代理。他的作用是返回给NavigationController一个Transition对象，用于执行转换效果。


2.进入TransitionFromFirstToSecond.m里

	- (NSTimeInterval)transitionDuration:(id<UIViewControllerContextTransitioning>)transitionContext {
	    return 0.3;
	}
	生成执行时间。

-(void)animateTransition:(id\<UIViewControllerContextTransitioning>)transitionContext 是主要操作的地方。
	
		//获得前后ViewController
	    FirstViewController *firstVC = (FirstViewController*)[transitionContext viewControllerForKey:UITransitionContextFromViewControllerKey];
    	SecondViewController *secondVC = (SecondViewController*)[transitionContext viewControllerForKey:UITransitionContextToViewControllerKey];
    	//获得执行时间
    	NSTimeInterval duration = [self transitionDuration:transitionContext];
		//containerView获得两个控制器里的view	
		UIView *containerView = [transitionContext containerView];
*  UITransitionContextFromViewControllerKey和UITransitionContextToViewControllerKey 分别获得前后ViewController

		//    获得第一个控制器的点击cell
	    ThingCell *cell = (ThingCell*)[firstVC.collectionView
                                         cellForItemAtIndexPath:[[firstVC.collectionView
                                                                  indexPathsForSelectedItems]
                                                                 firstObject]];
		//    获得cell的快照
		UIView *cellImageSnapshot = [cell.imageView snapshotViewAfterScreenUpdates:NO];
		//  获得cell里的imageView在整体view的位置
	    cellImageSnapshot.frame = [containerView convertRect:cell.imageView.frame
                                                fromView:cell.imageView.superview];
    
    	cell.imageView.hidden = YES;
    
    
		// 获得第二个viewController，隐藏imageview
	    secondVC.view.frame = [transitionContext finalFrameForViewController:secondVC];
	    secondVC.view.alpha = 0;
	    secondVC.imageView.hidden = YES;
    
    	[containerView addSubview:secondVC.view];
	    [containerView addSubview:cellImageSnapshot];

		[UIView animateWithDuration:duration animations:^{
        	// Fade in the second view controller's view
        	secondVC.view.alpha = 1.0;
        
	        // Move the cell snapshot so it's over the second view controller's image view
    	    //移动cell的快照倒第二个控制器的image view上
        	CGRect frame = [containerView convertRect:secondVC.imageView.frame fromView:secondVC.view];
	        cellImageSnapshot.frame = frame;
        
    	} completion:^(BOOL finished) {
	        //清理
    	    secondVC.imageView.hidden = NO;
	        cell.hidden = NO;
    	    [cellImageSnapshot removeFromSuperview];
        
        //声明转场结束,移除多余资源
        [transitionContext completeTransition:!transitionContext.transitionWasCancelled];
		}];


至此，第一个转场效果执行完毕。

下面看一下SecondViewController.m

在viewdidload里面多加一个平移的手势：

		UIScreenEdgePanGestureRecognizer *popRecognizer = [[UIScreenEdgePanGestureRecognizer alloc] initWithTarget:self                                                                                                       											action:@selector(handlePopRecognizer:)];
		    popRecognizer.edges = UIRectEdgeLeft;
    	[self.view addGestureRecognizer:popRecognizer];


这里实现手势函数
pragma mark UIGestureRecognizer handlers
			
	- (void)handlePopRecognizer:(UIScreenEdgePanGestureRecognizer*)recognizer {
    
    	CGFloat progress = [recognizer translationInView:self.view].x / (self.view.bounds.size.width * 1.0);
	    progress = MIN(1.0, MAX(0.0, progress));
	//    NSLog(@"%f",progress);
    
    	if (recognizer.state == UIGestureRecognizerStateBegan) {
        	// Create a interactive transition and pop the view controller
	        //创建一个交互对象，并pop控制器
    	    self.interactivePopTransition = [[UIPercentDrivenInteractiveTransition alloc] init];
        	[self.navigationController popViewControllerAnimated:YES];
        	
	    }else if (recognizer.state == UIGestureRecognizerStateChanged) {
        	// Update the interactive transition's progress
	        // 更新转场时的进度
    	    [self.interactivePopTransition updateInteractiveTransition:progress];
    	    
    	}else if (recognizer.state == UIGestureRecognizerStateEnded || recognizer.state == 			UIGestureRecognizerStateCancelled) {
        // Finish or cancel the interactive transition
        if (progress > 0.5) {
		//结束平移
            [self.interactivePopTransition finishInteractiveTransition];
        }else {
		//            取消平移
            [self.interactivePopTransition cancelInteractiveTransition];
        }
        
        	self.interactivePopTransition = nil;
    	}
    
	}

注意添加手势代理

	-(id<UIViewControllerInteractiveTransitioning>)navigationController:(UINavigationController *)navigationController
    	                     interactionControllerForAnimationController:							(id<UIViewControllerAnimatedTransitioning>)animationController {
    	                     
    		if ([animationController isKindOfClass:[TransitionFromeSecondToFirst class]]) {
			//返回手势的情况
    	    return self.interactivePopTransition;
		    }else {
    	    return nil;
    		}
	}


以上就是一个比较简单的转场效果，项目例子在附件里。里面有比较详细的注释，应该很容易理解。
