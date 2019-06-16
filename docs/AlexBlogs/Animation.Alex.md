
## 一.CoreAnimation介绍

CoreAnimation是一套图像渲染和动画基础框架，其在iOS和OSX平台用于显示对象和实现动画效果。使用CoreAnimation框架，动画的大部分帧渲染都是苹果为我们做好的。我们只需要配置几个动画参数（如开始和结束的点）并调用动画开始的方法。接下来就把剩余的工作交给CoreAnimation，操作全部实际绘图工作是在图形渲染硬件加速处理的。这个自动的图像加速器将会产生高帧频和顺滑的动画效果而不会加重CPU的负荷、或使APP卡顿。


CoreAnimation是在UIKit和APPKit框架之下，并且被很好的整合到Cocoa和Cocoa Touch的view中。同时CoreAnimation也给出了一些扩展的动画接口供我们使用。

下图是CoreAnimation在cocoa框架中的层级(图片来自苹果)
![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/5f2b2ad7ff941945d36342f702b9e5b3)

CoreAnimation
苹果对于CoreAnimation的介绍中首先讲述的是CALayer，因为CALayer是视图显示的基础、同时是CAAnimation动画产生的的载体。所有的动画都是作用在CALayer上，通过更改CALayer的属性，将每一帧渲染出来就形成了我们视觉的动画效果。但是这篇博客主要介绍CAAnimation，所以直接先忽略了前面的CALayer介绍，关于CALayer会在下一篇的文章中做详细介绍。

## 二.CoreAnimation动画

上面已经说过了，CoreAnimation是一套图形渲染与动画框架，CALayer负责图形的渲染显示，而CAAnimation及其子类则负责动画的实现。通过CAAnimatin及其子类我们能够相对简单的实现一些复杂的layer动画。

下面是整个CoreAnimation框架的所有动画类结构：
![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/8036be4f939205c5cb1430fba10a8b0a)

动画类结构
### 1.动画

CAAnimation是CoreAnimation的抽象超类、而CAPropertyAnimation又是CAAnimation的抽象子类，正如我们所知道的我们不应该直接使用抽象类，而应该使用它们的子类，如CABasicAnimation(基础动画)、CAKeyFrameAnimation(关键帧动画)、CASpringAnimation(弹簧动画)等。

#### 1.1 CAAnimation

CAAnimation是CoreAnimation的抽象超类，也是整个动画的核心类，大部分的动画属性与方法都是在该类中实现的。CAAnimation之所以能够拥有这些动画相关的属性和方法是因为该类遵守了CAMediaTiming、 CAAction两个协议。CAMediaTiming协议主要实现一些控制动画执行时间的属性和函数(包含动画的开始时间 -> beginTime、执行时间 -> duration、执行速率 -> speed、执行时间偏移量 -> timeOffset、重复执行的次数 -> repeatCount、动画执行结束的处理 ->filleMode等)，因此CAAnimation能够很好的处理时间与layer动画的关系；CAAction主要实现一些动作触发的响应接口，遵守该协议的对象可以指定CALayer响应的行为，如添加一个动画效果，或者执行其他的tasks。

timingFunction：属性用于设置动画执行的时间步调，创建该对象相对简单，可以直接使用kCAMediaTimingFunction系列宏指定动画执行的时间为`linear', `easeIn', `easeOut' and `easeInEaseOut'或者也可以使用贝塞尔曲线函数创建一个CAMediaTimingFunction对象贝塞尔曲线控制点获取

delegate：设置Animation的代理对象，这样我们可以获取动画执行过程的一些状态，包括动画的开始和结束，如果你只是想在结束时获得回调通知，也可以调用setCompletionBlock:函数，设置动画完成的block回调。

removedOnCompletion：动画执行结束是否移除动画，默认YES，但是该参数必须与fillMode = kCAFillModeForwards 或者 kCAFillModeBoth同时设置才能实现在动画结束不移除动画layer。

#### 1.2 CAPropertyAnimation

CAPropertyAnimation是CAAnimation的抽象子类，使用该类可以创建一个操作CALayer属性值的Animation对象。该类主要实现接口用于指定实现动画的CALayer的属性。

#### 1.3 CABasicAnimation

CABasicAnimation是CAPropertyAnimation的子类，该类实现了三个属性fromValue、byValue、toValue、用于描述一个单关键帧动画执行过程的三个属性值。

#### 1.4 CAKeyFrameAnimation

CAKeyFrameAnimation是CAPropertyAnimation的子类，该类用于实现多关键帧动画。我们可以将动画主要的一些帧值添加到数组，赋值给values属性，再把每一帧对应的时间添加到keyTimes属性，(值得注意的是keyTimes的值必须在【0，1】之间，数组中的值按照数组的index依次增大，并且所有值加在一起的和等于一)，如果想要为每一帧指定一个CAMediaTimingFunction对象，也可以创建对应的CAMediaTimingFunction对象并加到数组，然后赋值给timingFunctions属性。

CAKeyFrameAnimation最强大的地方在于他有一个path属性，当我们创建一个路径并赋值给path属性时，动画就会按照我们指定的路径轨迹执行。

#### 1.5 CASpringAnimation

CASpringAnimation继承于CABasicAnimation类，正如它的名字Spring一样，该类主要用于实现一些类似弹簧的动画效果。mass属性相当于物体的重量，stiffness属性代表了弹簧的刚度，damping属性代表阻尼系数，initialVelocity属性代表动画的初始速度，settlingDuration是动画的预估执行时间。通过这些属性值，我们可以控制Spring动画的拉伸幅度，动画的执行时间等。

### 2.组动画

在动画使用的过程中，我们一般不会只使用一个动画，我们可能为复杂的动画效果创建多个Animation对象，然后将这些对象分别添加到Layer动画中，这种方式能够实现但是相对比较麻烦，我们可以使用组动画解决这个问题。 对于组合动画我们可以使用CATransition()和CAAnimationGroup, CAAnimationGroup将多个动画合并为一组，并且我们可以指定动画关联时间让组内的动画可以同时或者按步骤执行，CATransaction将多个layer-tree更改操作放在一起执行并自动更新到render tree。

#### 2.1 CAAnimationGroup

CAAnimationGroup是CAAnimation的子类，该动画将多个动画放到一个animations数组，添加到layer，这些动画是并行执行的，当然你也可以设置例如beginTime这样的属性使动画按照一定的顺序执行。

#### 2.2CATransition()

CATransition继承于CAAnimation，该动画主要实现一些过渡效果。你可以为type属性设置`fade', `moveIn', `push' and `reveal'来实现你想要的动画效果，同时你也可以设置subtype为`fromLeft', `fromRight', `fromTop' and`fromBottom'来指定动画的方向。startProgress、endProgress用于控制开始和结束的进度，范围必须在【0，1】。

### 3.动画执行时间

时间是动画的重要部分之一，我们可以通过CAMediaTiming协议的方法和属性指定动画的时间信息。在CoreAnimation中遵守这个协议的有两个类。CAAnimation类采用了这个协议，因此可以为Animations执行指定时间信息。CALayer类也采用了该协议，因此也可以为隐式动画指定Animations执行相关的时间信息，值得注意的是隐式动画优先采用默认的时间信息。

想想时间和动画的关系，理解layer对象是如何和时间相互作用的是非常重要的。每一个layer对象都有一个本地的时间去管理动画时间。通常在动画过程中的两个layer的本地时间非常接近，接近到我们可能察觉不到有什么不同。本地时间可以被父layer或者自己拥有的timing参数更改

CALayer类定义了convertTime:fromLayer: 和convertTime:toLayer:方法，为了确保时间值适用于layer。我们可以使用这些方法将layer的时间值转换为相对于本地时间或者另一个layer时间值的精确时间。使用这些方法需要考虑到对layer的本地时间的影响，同时这些方法会返回一个可以在其他的layer中使用的值。

一旦我们有了一个layer的本地时间值，我们就可以使用这个值去更新Animation对象或者layer的相关属性，使用CAMediaTiming协议属性，我们可以实现一些有趣的动画效果，包括：

1.我们可以使用beginTime属性设置Animation的开始时间。通常情况，Animations的开始时间是在下一次更新循环(the next update cycle),但是我们可以设置beginTime参数来延迟动画的执行时间。我们可以设置后一个动画的beginTime为前一个动画的结束时间，这样多个动画按照一定的顺序执行。

2.使用timeOffset属性可以在一组动画中设置相对于其他动画，延迟执行一定的时间段。

4.动画的添加与删除

我们创建好动画对象可以调用CALayer的addAnimation:forKey:方法将动画添加到layer上并指定动画的标记key用于后续的删除操作，如果要移除一个动画可以调用CALayer的removeAnimationForKey:方法移除key值对应的动画效果、或者你可以调用removeAllAnimations方法删除layer附带的所有动画效果，并且remove方法是立即生效的，动画立即结束执行。

 ## 三.CoreAnimation动画实现方式

通过更改Layer的属性值实现动画。(属性必须是Animatable，关于CALayer的可做动画的属性)

在这里更改属性值创建动画有两种：

第一种是隐式动画(Animating a change implicitly),隐式动画使用默认的时间(0.25秒)和动画属性去执行动画。当我们更改layer的属性值时，会触发隐式动画。当修饰的layer对象在layer-tree中时，更新会立即执行。如果layer对象的显示效果没有立即改变，CoreAnimation会为这些改变创建一个触发器并且添加一个或者多个隐式动画去执行。因此，像theLayer.opacity = 0.0；这样的更改会引发CoreAnimation为你创建一个Animation对象，并把这个Animation对象加入到下一次更新循环(next update cycle.)中去执行。

第二种是显式动画(Animating a change explicitly)，需要你自己为使用的动画对象配置属性。如上的隐式动画如果要显示的执行，则需要我们创建一个Animation对象(如:CABasicAnimation)。并为这个对象配置动画参数。我们可以在把这个Animation添加到layer之前设置Animation的开始和结束值、动画的执行时间、或者设置其他的动画属性。当创建一个Animation对象，你需要指定动画的KeyPath并设置动画参数。去执行动画时，只需要调用addAnimation:forKey方法去将你想要执行的动画添加到layer上。显示动画的layer属性更新不像隐式动画，显示动画只是创建动画不会更改layer-tree中的属性值。在动画的结束CoreAnimation会移除动画并使用它当前的属性值重绘layer。如果你想要通过显示动画永久更改layer-tree中的属性值，那么你需要在动画结束时手动设置layer的属性。

隐式动画和显示动画通常是在当前的运行环(run loop)结束之后执行的，并且当前线程一定要有一个运行环去执行Animations。如果更改多个属性或者为layer添加Animations，那么这些改变或者添加的动画是并发执行的。当然也可以为每个动画设置特定的执行时间，具体的使用下面会有代码。

1.使用UIView的分类实现动画

尽管我们可以直接使用CAAnimation接口实现想要的动画效果，但是对于简单的动画使用CAAnimation还是需要额外的步骤，其实苹果已经为我们做了一些扩展，这些扩展在UIView的分类中实现，所以对于UIView自带的layer我们可以直接使用UIView去实现一些简单的动画。关于如何实现UIView自带的layer动画可以看这里How to Animate Layer-Backed Views.

下面是通过UIView动画分类接口实现的动画效果：

### 1.实现更改view透明度动画

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/ee365c9900eb9798ba0d4750aff8bcf4)

透明度
2.实现更换view背景色动画并延迟0.5秒执行

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/57ba0fc113c8c3684ebcda9f7e6ec3eb)

背景色
3.实现view移动动画

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/7d0e4b4973e7817d985df0150ba7c42c)

移动
4.实现view旋转动画

![Image of Yaktocat](++++++ 5 +++++++++)

旋转
5.实现view放大缩小动画

![Image of Yaktocat](++++++ 6 +++++++++)

放大

![Image of Yaktocat](++++++ 7 +++++++++)

缩小
6.实现view弹簧效果动画

![Image of Yaktocat](++++++ 8 +++++++++)

弹簧
7.实现view系统删除动画

![Image of Yaktocat](++++++ 9 +++++++++)

系统删除
8.实现过渡动画

![Image of Yaktocat](++++++ 10 +++++++++)

过渡
9.事务实现view翻转动画

![Image of Yaktocat](++++++ 11 +++++++++)

事务翻转
10.实现view组合动画

![Image of Yaktocat](++++++ 12 +++++++++)

UIView组合动画


### 2.使用CAAnimation实现动画

UIView自带的动画是苹果给我提供的CAAnimation的封装，其动画实现相对来说比较简单方便，我们只需要在block内更改UIView的属性便可以实现一些简单的动画效果。但是其局限性也是十分明显的。对于一些复杂动画、高级动画UIView自带的动画就显得无能为力，这时CAAnimation的强大之处就无语言表了。其实CAAniamtion动画的使用也不是很困难，但是其对于控制动画执行时间、数学知识运用以及超常的想象力等要求就比较高了。

下面是对照UIView自带动画接口实现的一些动画，并简单列举了几个动画效果，对于UIView自带动画来说实现相对困难，以此来展示下CAAnimation的强大。

1.实现更改view透明度动画

![Image of Yaktocat](++++++ 13 +++++++++)

透明度
2.实现更改view的背景色

![Image of Yaktocat](++++++ 14 +++++++++)

背景色
3.实现view移动动画

![Image of Yaktocat](++++++ 15 +++++++++)

移动
4.实现view曲线移动

![Image of Yaktocat](++++++ 16 +++++++++)

曲线移动
5.实现view旋转动画

![Image of Yaktocat](++++++ 17 +++++++++)

旋转
6.实现viewX轴翻转动画

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/56a3f9fe87fa6149050333dea8c3a762)

X轴翻转
7.实现viewY轴翻转动画

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/435d7466a7a9b16c83ed2d82412ddfc9)

Y轴翻转
8.实现view放大缩小动画

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/4d380ddf501d1fec945ff2f9d9c4db20)

放大

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/c4da04eefe2c5aafc1f6a13b4b503f2a)

缩小

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/46f4ad5c02da56c5585f49ea6f7e9ae3)

函数实现
9.实现view过渡动画

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/d3fc11c0b60da84f03c50ee0355574cd) <br/>过渡

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/7e92d8d38e2db3f595142aa4805e9420)

函数实现

10.实现view弹簧动画

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/a449c808d3c30498760c9f6ada4d968e)

弹簧动画

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/4388860465226e6d03d6941831d550cf)

函数实现
11.事务动画

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/0e95512e12e2731b81f8e28ff0e2161a)

事务
12.实现view组动画

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/f4a8f8e79870f53353a70ce8966e67c7)

组动画

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/2490bab3a375ba1e87a4f06dc59d2f59)

函数实现
## 四.自定义动画

上面说了那么多，但是创建的动画效果都还是比较普通的。对于CAAnimation类族，这些只是他们强大功能的冰山一角。但是无论多么复杂的动画其实都是上面这些基础动画的聚合，只要我么能够想象出动画的执行逻辑，我们就可以使用CAAnimation类族实现相对复杂的动画。本人的想象力有限，于是我模仿微博的tabbar中间item点击按钮的点击弹出菜单效果，实现了简单的弹出菜单，具体的代码：clone git代码。由于不能上传视频所以这里就不放效果图了，感兴趣的可以去下载工程运行试看。

## 五.总结

这篇博客主要介绍了CoreAnimation的动画使用，以及动画的实现。虽然CAAnimation类族的使用并不困难，但是还是有很多细节需要注意的。

1.显示动画在执行过程中的属性值修改并不会影响到layer-tree中的layer属性值，所以在动画执行结束，layer会回到原始状态。

2. 对于CALayer如果多次调用addAnimation: forKey:只会执行最后添加的Animation对象。

3. 对于一些属性值不是OC对象的需要将对应的结构体或者基本数据类型转化为OC对象，比较特殊的是颜色值，要将UIColor转化为CGColor并对其进行id强转。具体的属性值转换下表有对应的转换对象。

4. 其实动画中比较强大的属性是CALayer的transform属性，该属性可以实现3D动画效果，苹果也给出了一些transform的便捷keyPath，如rotation.x、scale.y、translation.z等。

5. 还有一点需要注意的是CAAnimation对象属性的设置必须在添加到CALayer之前，否则设置不起作用。

做为一个iOS菜鸟希望自己能够不断提高自己的撸码能力的同时，也给大家带来更多的贡献；马上就要十一了， 提前祝大家节日快乐、happy coding。

终于结束了，谢谢大家的阅读！觉得不错的朋友记得点个喜欢哦！有什么不足的希望大家评论指出，或者QQ我（1034131730）。

注：以下是一些小的细节

属性值得转换：

当我们创建动画更改的layer属性为C语言的结构体时，我们必须将这些结构体转换为一个对象赋值给layer下面的表列出了C语言类型对应的转换Obj-C对象。

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/3d36e19cbd34a87797f4adec1f1a3ad2)

动画属性值转换
CATransform3D的一些便捷的KeyPaths：

![Image of Yaktocat](https://user-gold-cdn.xitu.io/2017/9/28/9a81193bc5e06b3d1550184210d04b8d)

CATransform3D



