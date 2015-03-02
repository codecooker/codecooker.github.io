---
layout: post
title: Linux编译Objective-C 
category: objective-c
tags: [objective-c,linux,ubuntu]
---
{% highlight objective-c%}
#define kRotateAngle                                (M_PI/12)
#define kProjectionFactor                           (-0.005)
#define kScaleFactor                                (0.9f)
    
    UIView* showInView = self.superview;
    UIView* backgroundView = [showInView viewWithYYTag:kSheetBackgroundViewTag];
    if (!backgroundView && show) {
        backgroundView = [[UIView alloc]initWithFrame:showInView.bounds];
        backgroundView.backgroundColor = [UIColor blackColor];
        backgroundView.yyTag = kSheetBackgroundViewTag;
    }
    //做页面扭动动画
    UIImageView* showInViewImageView = (UIImageView*)[backgroundView viewWithYYTag:kSheetShowInViewImageViewTag];
    if (!showInViewImageView && show) {
        showInViewImageView = [[UIImageView alloc]init];
        showInViewImageView.image = [showInView viewScreenshot];
        showInViewImageView.frame = showInView.frame;
        showInViewImageView.yyTag = kSheetShowInViewImageViewTag;
        [backgroundView addSubview:showInViewImageView];
    }
    [showInView insertSubview:backgroundView belowSubview:self];
    CAAnimationGroup* animationGroup = [[CAAnimationGroup alloc]init];
    CAKeyframeAnimation* animation = [CAKeyframeAnimation animationWithKeyPath:@"transform"];
    CATransform3D transform = CATransform3DIdentity;
    transform.m34 = kProjectionFactor;
    transform = CATransform3DRotate(transform, kRotateAngle, 1, 0, 0);
    CATransform3D transform1 = CATransform3DIdentity;
    transform1.m34 = kProjectionFactor;
    CABasicAnimation* animationScale = [CABasicAnimation animationWithKeyPath:@"transform"];
    animationScale.duration = YYDefaultAnimationDuration * 2;
    CGFloat offestY = -20.0f;           //动画过程中将view向上提20px
    if (show) {
        animation.values = [NSArray arrayWithObjects:[NSValue valueWithCATransform3D:CATransform3DIdentity],[NSValue valueWithCATransform3D:transform],[NSValue valueWithCATransform3D:transform1] ,nil];
        animationScale.toValue = [NSValue valueWithCATransform3D:CATransform3DTranslate(CATransform3DMakeScale(kScaleFactor, kScaleFactor, 1), 0, offestY, 0)];
    }else{
        animation.values = [NSArray arrayWithObjects:[NSValue valueWithCATransform3D:CATransform3DTranslate(CATransform3DMakeScale(kScaleFactor, kScaleFactor, 1), 0, offestY, 0)],[NSValue valueWithCATransform3D:transform],[NSValue valueWithCATransform3D:transform1] ,nil];
        animationScale.toValue = [NSValue valueWithCATransform3D:CATransform3DTranslate(CATransform3DIdentity, 0, 0, 0)];
    }
    animation.keyTimes = [NSArray arrayWithObjects:[NSNumber numberWithFloat:0.0],[NSNumber numberWithFloat:0.4],[NSNumber numberWithFloat:1], nil];
    animation.duration = YYDefaultAnimationDuration * 2;
    [animationGroup setAnimations:[NSArray arrayWithObjects:animation,animationScale, nil]];
    animationGroup.duration = YYDefaultAnimationDuration * 2;
    animationGroup.removedOnCompletion = NO;
    animationGroup.fillMode = kCAFillModeForwards;
    animationGroup.timingFunction = [CAMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseIn];
    animationGroup.delegate = self;
    NSString* animationKey = show ? @"AnimationType2Show" : @"AnimationType2Hide";
    [showInViewImageView.layer addAnimation:animationGroup forKey:animationKey];
    [self showSheetViewWithAnimationType1:show];
{% endhighlight %}