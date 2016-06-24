# Coordinate Quartz UIKit

坐标系一直是我头痛的问题，在多年以前看Quartz 2D Programming Guide的时候就比较懵逼，当时貌似研究了一阵子似乎是懂了但是其实应该还是云里雾里吧，至于我后来都忘了当时的思考与结论。坐标系复杂是因为UIKit和Quartz中的API使用的是不同的坐标系。UIKit零点在左上角，Y轴向下增长；Quartz的零点是在左下角，Y轴向上增长，所以使用Quartz的api的时候都要做坐标变换。

首先说说坐标系的作用，苹果在文档中说的听清楚了，作用有两个：
1. 第一个开发方便。坐标系增加了一层抽象，我们平时开发的时候坐标单位都是point，实际苹果绘制图片的时候都是以像素为单位，但是因为设备不同，一个点对应的像素是不同的，iPhone3gs是一个点对应1个像素，而iPhone4是2*2个像素，如果我们写代码的时候使用的是像素，那就麻烦很多，比如我们要画一条线到屏幕中间，那在3gs上就是画到160位置，而4上就是320的位置。总之使用point隐藏了设备的分辨率，我们的代码可以在同样宽高比的其他设备上也照样使用，只是内部的point对应的像素数量改变了。
2. 第二个是我们可以对坐标系做变换，然后实现绘制的变换。比如让坐标系转45度，那么图片也就相应的旋转了。

我们绘制的目标有很多，可能是屏幕、打印机，也可能是位图。
不同的目标使用的坐标系是不同的，原因是它们本身的坐标系约定不同，在iOS中屏幕表示的是UIView，它的坐标系是向下增加，而CGBitmapContext是向上增加，我们以最为熟悉的UIKit来说明。我们向屏幕绘制的方法是覆盖UIView的`-drawRect:`方法，然后使用`UIGraphicsGetCurrentContext`获取当前的context，使用context调用quartz的方法进行绘制。如果利用`CGContextGetCTM(context)`方法拿到此时坐标系的变换矩阵，可以看到结果是[2, 0, 0, -2, 0, 1206]，这个矩阵是UIKit在内部做变换产生的，目的我们在下面会说明，在我们调用Quartz API进行绘制时，内部会应用这个矩阵。我的测试机器是iPhone6，这个`UIView`的`size`是375 * 603 point，转换成像素是750 * 1206，scale是2.0(在6 plush上是3.0)。

我想说明一下这个矩阵起到的作用，比如我想绘制一条线，从(0, 0)到(100, 100)，我要调用下面的方法:
```
CGContextMoveToPoint(context, 0, 0);
CGContextAddLineToPoint(context, 100, 100);
```
这两个方法都是Quartz的方法，而Quartz的坐标系是左下角为零点，另外，Quartz的坐标系的变换矩阵是[2, 0, 0, 2, 0, 0]，因为绘制的目标的单位是像素，point只是我们抽象出来的单位，所以要利用这个矩阵将点转换成像素，即x和y都乘以2。

如果不应用变换矩阵，在Quartz上绘制(0,0)到(100,100)的线，会以左下角开始做一条线，终点是向上100个point，向左100个point，这条线在UIView(375*603)的坐标系上是从(0,603)到(100,503)的线，而我们希望绘制的是从(0,0)到(100,100)的线。

别忘记我们在绘制之前要先应用矩阵变换，我们将(0,0) (100,100)应用之前拿到的变换矩阵[2, 0, 0, -2, 0, 1206]，拿到的结果是(0,1206), (200,1006)，这个结果的单位是像素，转换成point是(0,603),(100, 503)，这两个点在Quartz的坐标系中正好是UIKit坐标系中(0,0) (100,100)的位置。这就是这个变换矩阵存在的意义了，它能将我们在UIKit坐标系中的点转换到Quartz坐标系上并保持点的位置不变。

呃，很绕吧？如果你还可以听懂，那么我们继续，刚刚说的是画线，现在我们要画图了。在UIKit中画图我们可以使用`UIImage`的绘制方法，例如`drawRect:`/`drawAtPoint:`，还可以使用Quartz的`CGContextDrawImage`，我现在想解释一下为什么在UIKit的context中要使用`UIImage`的绘制方法。

要说明为什么要使用`UIImage`的绘制方法，就先说明为什么不使用`CGContextDrawImage`，那么做个图试试不就知道了么，下面的代码会画一个图：
`CGContextDrawImage(context, CGRectMake(0, 0, imageWidth, imageHeight), image.CGImage);`
这个方法会在`UIView`的左上角绘制图片，关于这个位置的变换我刚刚讨论过了，可以肯定确实图片出现在了那里！但是等等，你会发现图片是倒过来的！！！为什么会这样？

考虑绘制过程，我们使用的是Quartz的方法绘制的，所以绘制是在Quartz的坐标系进行的，假如这个图是人的肖像图，那么y轴为0的这个位置大约是下巴这里，而y轴在imageHeight的位置大约是头发的位置，别忘记矩阵变换，在绘制前应用了矩阵变换，那么Quartz上y轴为0在变换后会变成1006这个位置，而y轴为imageHeight（point为200，pixel为400）则变成了606，所以头发在下面，下巴在上面，图片当然就倒过来了。

那么UIImage的方法为什么可以正确绘制？我觉得在其内部进行了矩阵变换，我用代码模拟了一下。
```
CGContextSaveGState(context);
// 恢复成Quartz的矩阵变换
CGContextScaleCTM(context, 1.0, -1.0);
CGContextTranslateCTM(context, 0, -self.size.height);
CGContextDrawImage(context, CGRectMake(0, self.size.height-image.size.height, image.size.width, image.size.height), image.CGImage);
CGContextRestoreGState(context);
```
代码的思路是将UIKit的矩阵变换恢复成Quartz的矩阵变换，然后在Quartz中计算位置来绘制图片。那么我怎么就知道要这么做变换呢？那我就要说说线性代数的知识了。

变换矩阵是3*3的矩阵，[a, b, 0, c, d, 0, tx, ty, 1]，这里不方便写成矩阵，我就按照从每行开始向后排列的顺序列出来这个矩阵。一个点(x, y)应用了矩阵变化之后变成了(x', y')，公式是下面这样：(x', y', 1) = (x, y, 1) * [a, b, 0, c, d, 0, tx, ty, 1]
```
x' = a * x + c * y + tx;
y' = b * x + d * y + ty;
```

在我们上面绘制图片的代码中，`UIKit`的`drawRect:`方法中拿到的context的变换矩阵是[2, 0, 0, -2, 0, 1206]，将这个矩阵应用矩阵变换`CGContextScaleCTM(context, 1.0, -1.0);`之后的结果是[2, 0, 0, 2, 0, 1206]，
因为文档中没有说明这个方法内部是如何实现的矩阵变换，我们从结果推导过程，从结果来看应该是矩阵[2, 0, 0, -2, 0, 1206] * [1, 0, 0, 0, -1, 0, 0, 0, 1]得到的值；

接着我们再次应用矩阵变换`CGContextTranslateCTM(context, 0, -self.size.height);`，打印出结果是[2, 0, 0, 2, 0, 0]，继续推导过程是：[2, 0, 0, 2, 0, 1206] * [1, 0, 0, 0, 1, 0, 0, -1206, 1]这两个矩阵相乘得到的。

由此可知，我将context中的变换矩阵表示成transformMatrix:
`CGContextScaleCTM(context, 1.0, -1.0)` 是 transformMatrix = transformMatrix * [1, 0, 0, 0, -1, 0, 0, 0, 1]；
`CGContextTranslateCTM(context, 0, -self.size.height);` 是 transformMatrix = [1, 0, 0, 0, 1, 0, 0, -1206, 1] * transformMatrix；

为什么`CGContextScaleCTM`中原来的矩阵是在乘号的左边，而在`CGContextTranslateCTM`中则是右边？`CGContextScaleCTM`的目的是更改scale的值，即将结果做乘法，而`CGContextTranslateCTM`的目的是更改tx和ty，即对结果做加法。

假如调用`CGContextScaleCTM(context, Sx, Sy)`，原来的矩阵是[a, b, 0, c, d, 0, tx, ty, 1]，方法调用完后得到矩阵[a*Sx, b*Sy, 0, c*Sx, d*Sy, 0, tx*Sx, ty*Sy, 1]，之前x'和y'的公式则变成了
```
x' = a*Sx * x + c*Sx * y + tx*Sx;
y' = b*Sy * x + d*Sy * y + ty*Sy;
```
提出Sx和Sy，则变成：
```
x' = Sx(a * x + c * y + tx);
y' = Sy(b * x + d * y + ty);
```
我们继续分析`CGContextTranslateCTM`：

假如调用`CGContextTranslateCTM(context, Tx, Ty)`，原来的矩阵还是[a, b, 0, c, d, 0, tx, ty, 1]，方法调用完后得到矩阵[a, b, 0, c, d, 0, a*Tx+c*Ty+tx, b*Tx+d*Ty+ty, 1]，之前x', y'的公式变成了
```
x' = a * x + c * y + a*Tx+c*Ty+tx;
y' = b * x + d * y + b*Tx+d*Ty+ty;
```
确实也只改变了平移的部分。如果有什么不明白的地方请前去阅读线性代数，如有问题请联系我。
