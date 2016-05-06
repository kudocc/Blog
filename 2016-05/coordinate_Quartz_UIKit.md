# Use Quartz function in UIKit Coordinate

UIKit的坐标系是Top-Left，向下和向右增加；Quartz的坐标系是Bottom-Left，向上向右增加。在UIView的函数drawRect:和UIGraphicsBeginImageContextWithOptions中，调用`UIGraphicsGetCurrentContext`拿到的context是Top-Left的，这是由于UIKit提前把这个context做了翻转。可以通过CGContextGetCTM(context)获取当前的CTM，我打印出来结果是：[2, 0, 0, -2, 0, 1334]，我的设备是6s，所以scale是2，可以看出UIKit提前将坐标系向上移动了567并且对Y轴做了翻转。现在我们开始在这里坐标系中画图，我的代码是这样的：

```
- (void)drawRect:(CGRect)rect {
    CGFloat y = 64;
    [image drawInRect:CGRectMake(0, y, image.size.width, image.size.height)];
    
    CGContextRef context = UIGraphicsGetCurrentContext();
    y += image.size.height;
    CGContextDrawImage(context, CGRectMake(0, y, image.size.width, image.size.height), image.CGImage);
    
    // flip y
    y += image.size.height;
    CGContextTranslateCTM(context, 0, self.bounds.size.height);
    CGContextScaleCTM(context, 1, -1);
    CGContextDrawImage(context, CGRectMake(0, 0, image.size.width, image.size.height), image.CGImage);
}
```

我画了三张图，第一个图使用UIImage的`drawInRect:`方法，第二个图在第一个图的下面，使用CGContextDrawImage方法，在画第三个图之前，我把坐标系变回了Quartz的默认坐标系（[2, 0, -0, 2, 0, 0]）。结果是第二个图是相对于原图，翻转过来了，第一个图和第三个图都正确的绘制出来。第二个图翻转的原因是UIKit的坐标系变换，Quartz还是按照原本的方式绘制，像素点，按照Quartz坐标系的从0到image.size.width，再从0到image.size.height。想象此时如果坐标系没有变换，图片绘制的位置和方向，然后按照坐标系变换的流程，想象最后图片会怎样，是翻转了的！那么有一个疑问，为什么UIKit的方法drawInRect:绘制的图片是正常的呢？UIKit也是调用的CGContextDrawImage，但是它在内部做了一次坐标系变换，现在我们来模拟一次，修改代码如下：

```
- (void)drawRect:(CGRect)rect {
    CGFloat y = 64;
    [image drawInRect:CGRectMake(0, y, image.size.width, image.size.height)];
    
    CGContextRef context = UIGraphicsGetCurrentContext();
    y += image.size.height;
    CGContextSaveGState(context);
    CGContextTranslateCTM(context, 0, y+image.size.height);
    CGContextScaleCTM(context, 1.0, -1.0);
    CGContextDrawImage(context, CGRectMake(0, 0, image.size.width, image.size.height), image.CGImage);
    CGContextRestoreGState(context);
    
    // flip y
    y += image.size.height;
    CGContextTranslateCTM(context, 0, self.bounds.size.height);
    CGContextScaleCTM(context, 1, -1);
    CGContextDrawImage(context, CGRectMake(0, 0, image.size.width, image.size.height), image.CGImage);
}
```

再次运行，第二个图片也正常的绘制出来了。

所以在drawRect:中使用Quartz的方法绘制时会被翻转，我们这里要么自己做坐标系的转换，要么使用UIKit的方法。
