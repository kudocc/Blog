# Coordinate

坐标系一直是我头痛的问题，在多年以前看Quartz 2D Programming Guide的时候就比较懵逼，当时貌似研究了一阵子似乎是懂了但是其实应该还是云里雾里吧，至于我后来都忘了当时的思考与结论。坐标系复杂是因为UIKit和Quartz中的API使用的是不同的坐标系。UIKit零点在左上角，Y轴向下增长；Quartz的零点是在左下角，Y轴向上增长，所以使用Quartz的api的时候都要做坐标变换。

首先说说坐标系的作用，苹果在文档中说的听清楚了，作用有两个：
1. 第一个开发方便。坐标系增加了一层抽象，我们平时开发的时候坐标单位都是point，实际苹果绘制图片的时候都是以像素为单位，但是因为设备不同，一个点对应的像素是不同的，iPhone 6s是一个点对应3*3个像素，而6是2*2个像素，如果我们写代码的时候使用的是像素，那就麻烦了，所有的设备我们都要考虑一遍。
2. 第二个是我们可以对坐标系做变换，然后实现绘制的变换。比如让坐标系转45度，那么图片也就相应的旋转了。

我们绘制的目标有很多，可能是屏幕、打印机，也可能是位图。
我们像屏幕绘制的方法是覆盖UIView的`-drawRect:`方法，然后使用`UIGraphicsGetCurrentContext`获取当前的context，使用context调用quartz的方法进行绘制。如果利用`CGContextGetCTM(context)`方法拿到此时坐标系的变换矩阵，可以看到结果是[2, 0, 0, -2, 0, 1206]，在我们调用Quartz API进行绘制时，内部会应用这个矩阵。我的测试机器是iPhone6，UIView是375 * 603 point，转换成像素是750 * 1206，scale是2.0(在6 plush上是3.0)。

我想说明一下这个矩阵起到的作用，比如我想绘制一条线，从(0, 0)到(100, 100)，我要调用下面的方法:
```
CGContextMoveToPoint(context, 0, 0);
CGContextAddLineToPoint(context, 100, 100);
```
这两个方法都是Quartz的方法，而Quartz的坐标系是左下角为零点，另外，Quartz的坐标系的变换矩阵是[2, 0, 0, 2, 0, 0]，因为绘制的目标的单位是像素，点只是我们抽象出来的单位，所以要将点转换成像素，即x和y都乘以2。

如果不应用变换矩阵，在Quartz上绘制(0,0)到(100,100)的线，会以左下角开始做一条线，终点是向上100个point，向左100个point，这条线在UIView(375 * 603)的坐标系上是从(0,603)到(100,503)的线，而我们希望绘制的是从(0,0)到(100,100)的线。那么如果要将这两个点正确绘制在UIView的坐标系中的对应位置上，对应的Quartz上是哪两个点呢？我简单的计算了一下，它们的位置是(0, 603) (100, 503)，我们将它们都转换成像素，就变成了(0, 1206) (200, 1006)。

那么如果我们将(0, 0) (100, 100)应用之前拿到的变换矩阵[2, 0, 0, -2, 0, 1206]会如何呢？拿到的结果是(0, 1206), (200, 1006)，正好对应上了上面计算的值，这也说明了为什么应用这个变换矩阵可以正确的绘制了。

呃，很绕吧？如果你还可以听懂，那么我们继续，刚刚说的是画线，现在我们要画图了。在UIKit中画图我们可以使用`UIImage`的绘制方法，例如`drawRect:`/`drawAtPoint:`，还可以使用Quartz的`CGContextDrawImage`，我现在想解释一下为什么在UIKit的context中要使用`UIImage`的绘制方法。

要说明为什么要使用`UIImage`的绘制方法，就先说明为什么不使用`CGContextDrawImage`，那么做个图试试不就知道了么：
`CGContextDrawImage(context, CGRectMake(0, 0, imageWidth, imageHeight), image.CGImage);`
这个方法会在`UIView`的左上角绘制图片，关于这个位置的变换我刚刚讨论过了，可以肯定确实图片出现在了那里！但是等等，你会发现图片是倒过来的！！！为什么会这样？

考虑绘制过程，绘制是在Quartz的坐标系进行的，假如这个图是人的肖像图，那么y轴为0的这个位置大约是下巴这里，而y轴在imageHeight的位置大约是头发的位置，别忘记矩阵变换，在回之前应用了矩阵变换，那么Quartz上y轴为0在变换后会变成1006这个位置，而y轴为imageHeight（point为200，pixel为400）则变成了606，所以头发在下面，下巴在上面，图片当然就倒过来了。

那么UIImage的方法为什么可以正确绘制？我觉得在其内部进行了矩阵变换，我用代码模拟了一下。
```
    CGContextSaveGState(context);
    // 恢复成Quartz的矩阵变换
    CGContextScaleCTM(context, 1.0, -1.0);
    CGContextTranslateCTM(context, 0, -self.size.height);
    CGContextDrawImage(context, CGRectMake(0, self.size.height-image.size.height, image.size.width, image.size.height), image.CGImage);
    CGContextRestoreGState(context);
```
代码的思路是将UIKit的矩阵变换恢复成Quartz的矩阵变换，然后在Quartz中计算位置来绘制图片。
