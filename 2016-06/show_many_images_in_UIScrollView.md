# Show many images in UIScrollView

这个问题跟随我有好长时间了，在做IM聊天界面的时候就遇到这个问题，在聊天时发送图片消息很多的时候，迅速滑动UITableView会有卡顿现象发生，最开始卡顿还不是很明显，但是在需求的同事要求将图片消息加入圆角效果后，这件事情就没有办法回避了。

加入让你来实现一个展示图片消息的cell，图片需要带圆角，相信最直观的想法就是直接将图片放到`UIImageView`中，然后设置其layer的cornerRadius和masksToBounds属性，但是这样做就导致了刚刚提到的问题，卡顿，滑动时严重掉帧。

我想在给出所谓的best practice之前，先给出几个例子来看一下这些不同的方法在滚动时的帧率。

我使用的机器是我的iPhone4，系统是7.1.2，之所以使用旧设备是为了让效果更明显，测试帧率的方法是使用Profiling中的Core Animation。

我先准备了11张图片，图片有大有小，最小的也要超过150 * 150像素，最大的可能超过1000 * 768像素。我会创建一个UITableView，在每个cell中放置三张相同的图片，之所以使用三张图片而不是一张也是为了让测试效果更加明显，numberOfRows设置成了400，为了能尽情的滚动，方便检测FPS。每个图片的宽度*高度是66*66 point，它们都是小于图片的尺寸的。

我的测试分为两个大分类，第一个分类是在UIView上绘制图片，第二个分类是使用UIImageView。

每个分类都有4种不同的case，分别是 1.使用原图；2. 事先将图片大小变成66*66，再进行绘制；3. 事先将图片大小变成66*66，然后设置UIView的layer.cornerRadius和masksToBounds来实现圆角效果；4. 事先处理图片，将图片大小变成66*66并且clip掉圆角区域，即使修改图片本身使其带有圆角。

** 在UIView上绘制图片

这部分代码可以在[这里](https://github.com/kudocc/DemoKit/blob/master/demo/performance/DrawViewContainer.m)看到。

*** 1. 使用原图

得到的FPS实在可怜，只有不到10。

*** 2. 裁剪图片到66*66

FPS可以达到55以上，效果还是不错的，毕竟使用drawRect主要是耗费CPU的资源，而iPhone4的CPU是现在市面上最差的。

*** 3. 裁剪图片到66*66，并设置layer.cornerRadius和masksToBounds

FPS基本上在23上下，效果很差，之所以会导致帧率下降的如此厉害主要是这两个属性会导致离屏渲染

*** 4. 裁剪图片到66*66，clip掉图片的圆角区域

FPS在55以上，可以想象得到其与第二个case是一样的，他们都是66*66的图片，只是图片的内容略有不同而已。

** 设置UIImageView.image属性

这部分代码在[这里](https://github.com/kudocc/DemoKit/blob/master/demo/performance/ImageViewContainer.m)

*** 1. 使用原图

与想象中可能不太一样，FPS可以达到55以上，在55-59之间，可以看出UIImageView不仅仅是把UIImage绘制上去这么简单，它一定做了很多优化，所以能使用UIImageView就尽量使用它吧！

*** 2. 裁剪图片到66*66

与case差不多，我觉得UIImageView内部其实也将图片做了裁剪吧，如果让我自己来实现UIImageView的话，因为图片可能远远大于UIImageView的bounds.size，在bounds之外的内容不展示的情况下，我想也没有必要把整个图片都提交到GPU吧。

*** 3. 裁剪图片到66*66，并设置layer.cornerRadius和masksToBounds

FPS也是23左右，看来UIImageView拿离屏渲染也没辙。

*** 4. 裁剪图片到66*66，clip掉图片的圆角区域

FPS在55-59之间，滑动时非常流畅。

总结：当需要在ScrollView中展示许多图片的时候，我们要考虑滚动内容的流畅性，此时我们尽量先处理一下图片，除非你能够保证拿到的图片与你展示图片的容器一样大，不过即使这样，如果你需要圆角效果，那么还是建议你在后台处理图片，然后再绘制或者放到UIImageView中，从我们的讨论中可以看出，UIImageView做了一些优化，所以这里推荐使用UIImageView。

还要提到的一点是我们提供的图片一般是PNG或者是JPEG格式的，他们都是一种图片的压缩格式，所以在绘制图片之前是需要被解压缩的，解压缩的过程也是一个消耗时间的过程，不过如果你能够像我说的那样在后台处理图片，那么你已经在后台解压缩并且处理好图片了，此时这个问题就不会困扰到你。
