---
title: 利用Core Graphics实现刮奖效果
date: 2017-04-10 16:53:00
tags: Objective-C
---

刮奖是商家类项目中经常使用的组件，其实现方式也有多种，下面介绍一种使用Core Graphics实现的一种方式。

Core Graphics中有一种根据遮罩图片(Masking Images)和原图片最终合成位图的方法，下面看一个官方文档给出的效果来一个直观的展示。
![原图](http://upload-images.jianshu.io/upload_images/1513759-1fd246bfd247ca29.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![遮罩图](http://upload-images.jianshu.io/upload_images/1513759-a551b423e34ebc65.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![合成图](http://upload-images.jianshu.io/upload_images/1513759-e83b5cbb73ba007d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从中可以看到黑色的部分将原图显示了出来，白色的部分把原图遮住了，灰色的部分和原图经过一定的算法进行了合成。我们可以通过不断的改变遮罩层中某部分的颜色，最终产生刮奖的效果。具体步骤如下：
![实现步骤](http://upload-images.jianshu.io/upload_images/1513759-60f70dd8905db127.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) 
从中可以看出，刚开始产生的Marsk是黑色的，这时合成之后蒙层图片原样展示，手指一动的时候往mask上绘制了白色的线条，这样，合成之后蒙层上被划过的地方被白色所取代，这样就出现了刮奖的效果，在每次绘制完成之后只需要调用`setNeedsDisplay`方法，然后在`drawRect`方法中不断展现最终合成的图片即可。
具体的代码如下:

```objc
CGColorSpaceRef colorspace = CGColorSpaceCreateDeviceGray();
float scale = [UIScreen mainScreen].scale;
//1. 获取刮奖层
UIGraphicsBeginImageContextWithOptions(hideView.bounds.size, NO, 0);
[hideView.layer renderInContext:UIGraphicsGetCurrentContext()];
hideView.layer.contentsScale = scale;
hideImage = UIGraphicsGetImageFromCurrentImageContext().CGImage;
UIGraphicsEndImageContext();

size_t imageWidth = CGImageGetWidth(hideImage);
size_t imageHeight = CGImageGetHeight(hideImage);
CFMutableDataRef pixels = CFDataCreateMutable(NULL, imageWidth * imageHeight);
//2. 获取context手指滑动时不断在这个context上画上白线。
contextMask = CGBitmapContextCreate(CFDataGetMutableBytePtr(pixels), imageWidth, imageHeight , 8, imageWidth, colorspace, kCGImageAlphaNone);
CGContextFillRect(contextMask, self.frame);

// 设置滑动时候产生的线条颜色是白色
CGContextSetStrokeColorWithColor(contextMask, [UIColor whiteColor].CGColor);
CGContextSetLineWidth(contextMask, _sizeBrush);
CGContextSetLineCap(contextMask, kCGLineCapRound);

CGDataProviderRef dataProvider = CGDataProviderCreateWithCFData(pixels);
CGImageRef mask = CGImageMaskCreate(imageWidth, imageHeight, 8, 8, imageWidth, dataProvider, nil, NO);

//2. 根据iamge mask产生最终的图片
scratchImage = CGImageCreateWithMask(hideImage, mask);
CGImageRelease(mask);
CGColorSpaceRelease(colorspace);
```

手指滑动时候调用的方法:

```objc
- (void)touchesMoved:(NSSet *)touches withEvent:(UIEvent *)event {
[super touchesMoved:touches withEvent:event];
UITouch *touch = [[event touchesForView:self] anyObject];
currentTouchLocation = [touch locationInView:self];
previousTouchLocation = [touch previousLocationInView:self];
[self scratchTheViewFrom:previousTouchLocation to:currentTouchLocation];
}

// 绘制图像
- (void)scratchTheViewFrom:(CGPoint)startPoint to:(CGPoint)endPoint {

BOOL needRender = [self needRenderWithCurrentLocation:endPoint previousLocation:previousTouchLocation];
if (!needRender) return;

float scale = [UIScreen mainScreen].scale;
CGContextMoveToPoint(contextMask, startPoint.x * scale, (self.frame.size.height - startPoint.y) * scale);
CGContextAddLineToPoint(contextMask, endPoint.x * scale, (self.frame.size.height - endPoint.y) * scale);
CGContextStrokePath(contextMask);
// 调用drawRect 方法
[self setNeedsDisplay];
self.isDrawn = YES;
}
- (void)drawRect:(CGRect)rect {
UIImage *imageToDraw = [UIImage imageWithCGImage:scratchImage];
[imageToDraw drawInRect:CGRectMake(0.0, 0.0, self.frame.size.width, self.frame.size.height)];
}
```