# 11. 基于定时器的动画

> _我可以指导你，但是你必须按照我说的做。_ -- 骇客帝国

在第10章“缓冲”中，我们研究了`CAMediaTimingFunction`，它是一个通过控制动画缓冲来模拟物理效果例如加速或者减速来增强现实感的东西，那么如果想更加真实地模拟物理交互或者实时根据用户输入修改动画改怎么办呢？在这一章中，我们将继续探索一种能够允许我们精确地控制一帧一帧展示的基于定时器的动画。

## 定时帧

动画看起来是用来显示一段连续的运动过程，但实际上当在固定位置上展示像素的时候并不能做到这一点。一般来说这种显示都无法做到连续的移动，能做的仅仅是足够快地展示一系列静态图片，只是看起来像是做了运动。

我们之前提到过iOS按照每秒60次刷新屏幕，然后`CAAnimation`计算出需要展示的新的帧，然后在每次屏幕更新的时候同步绘制上去，`CAAnimation`最机智的地方在于每次刷新需要展示的时候去计算插值和缓冲。

在第10章中，我们解决了如何自定义缓冲函数，然后根据需要展示的帧的数组来告诉`CAKeyframeAnimation`的实例如何去绘制。所有的Core Animation实际上都是按照一定的序列来显示这些帧，那么我们可以自己做到这些么？

### `NSTimer`

实际上，我们在第三章“图层几何学”中已经做过类似的东西，就是时钟那个例子，我们用了`NSTimer`来对钟表的指针做定时动画，一秒钟更新一次，但是如果我们把频率调整成一秒钟更新60次的话，原理是完全相同的。

我们来试着用`NSTimer`来修改第十章中弹性球的例子。由于现在我们在定时器启动之后连续计算动画帧，我们需要在类中添加一些额外的属性来存储动画的`fromValue`，`toValue`，`duration`和当前的`timeOffset`（见清单11.1）。

清单11.1 使用`NSTimer`实现弹性球动画

```text
@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *containerView;
@property (nonatomic, strong) UIImageView *ballView;
@property (nonatomic, strong) NSTimer *timer;
@property (nonatomic, assign) NSTimeInterval duration;
@property (nonatomic, assign) NSTimeInterval timeOffset;
@property (nonatomic, strong) id fromValue;
@property (nonatomic, strong) id toValue;

@end

@implementation ViewController

- (void)viewDidLoad
{
    [super viewDidLoad];
    //add ball image view
    UIImage *ballImage = [UIImage imageNamed:@"Ball.png"];
    self.ballView = [[UIImageView alloc] initWithImage:ballImage];
    [self.containerView addSubview:self.ballView];
    //animate
    [self animate];
}

- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    //replay animation on tap
    [self animate];
}

float interpolate(float from, float to, float time)
{
    return (to - from) * time + from;
}

- (id)interpolateFromValue:(id)fromValue toValue:(id)toValue time:(float)time
{
    if ([fromValue isKindOfClass:[NSValue class]]) {
        //get type
        const char *type = [(NSValue *)fromValue objCType];
        if (strcmp(type, @encode(CGPoint)) == 0) {
            CGPoint from = [fromValue CGPointValue];
            CGPoint to = [toValue CGPointValue];
            CGPoint result = CGPointMake(interpolate(from.x, to.x, time), interpolate(from.y, to.y, time));
            return [NSValue valueWithCGPoint:result];
        }
    }
    //provide safe default implementation
    return (time < 0.5)? fromValue: toValue;
}

float bounceEaseOut(float t)
{
    if (t < 4/11.0) {
        return (121 * t * t)/16.0;
    } else if (t < 8/11.0) {
        return (363/40.0 * t * t) - (99/10.0 * t) + 17/5.0;
    } else if (t < 9/10.0) {
        return (4356/361.0 * t * t) - (35442/1805.0 * t) + 16061/1805.0;
    }
    return (54/5.0 * t * t) - (513/25.0 * t) + 268/25.0;
}

- (void)animate
{
    //reset ball to top of screen
    self.ballView.center = CGPointMake(150, 32);
    //configure the animation
    self.duration = 1.0;
    self.timeOffset = 0.0;
    self.fromValue = [NSValue valueWithCGPoint:CGPointMake(150, 32)];
    self.toValue = [NSValue valueWithCGPoint:CGPointMake(150, 268)];
    //stop the timer if it's already running
    [self.timer invalidate];
    //start the timer
    self.timer = [NSTimer scheduledTimerWithTimeInterval:1/60.0
                                                  target:self
                                                selector:@selector(step:)
                                                userInfo:nil
                                                 repeats:YES];
}

- (void)step:(NSTimer *)step
{
    //update time offset
    self.timeOffset = MIN(self.timeOffset + 1/60.0, self.duration);
    //get normalized time offset (in range 0 - 1)
    float time = self.timeOffset / self.duration;
    //apply easing
    time = bounceEaseOut(time);
    //interpolate position
    id position = [self interpolateFromValue:self.fromValue
                                     toValue:self.toValue
                                  time:time];
    //move ball view to new position
    self.ballView.center = [position CGPointValue];
    //stop the timer if we've reached the end of the animation
    if (self.timeOffset >= self.duration) {
        [self.timer invalidate];
        self.timer = nil;
    }
}

@end
```

很赞，而且和基于关键帧例子的代码一样很多，但是如果想一次性在屏幕上对很多东西做动画，很明显就会有很多问题。

`NSTimer`并不是最佳方案，为了理解这点，我们需要确切地知道`NSTimer`是如何工作的。iOS上的每个线程都管理了一个`NSRunloop`，字面上看就是通过一个循环来完成一些任务列表。但是对主线程，这些任务包含如下几项：

* 处理触摸事件
* 发送和接受网络数据包
* 执行使用gcd的代码
* 处理计时器行为
* 屏幕重绘

当你设置一个`NSTimer`，他会被插入到当前任务列表中，然后直到指定时间过去之后才会被执行。但是何时启动定时器并没有一个时间上限，而且它只会在列表中上一个任务完成之后开始执行。这通常会导致有几毫秒的延迟，但是如果上一个任务过了很久才完成就会导致延迟很长一段时间。

屏幕重绘的频率是一秒钟六十次，但是和定时器行为一样，如果列表中上一个执行了很长时间，它也会延迟。这些延迟都是一个随机值，于是就不能保证定时器精准地一秒钟执行六十次。有时候发生在屏幕重绘之后，这就会使得更新屏幕会有个延迟，看起来就是动画卡壳了。有时候定时器会在屏幕更新的时候执行两次，于是动画看起来就跳动了。

我们可以通过一些途径来优化：

* 我们可以用`CADisplayLink`让更新频率严格控制在每次屏幕刷新之后。
* 基于真实帧的持续时间而不是假设的更新频率来做动画。
* 调整动画计时器的`run loop`模式，这样就不会被别的事件干扰。

### `CADisplayLink`

`CADisplayLink`是CoreAnimation提供的另一个类似于`NSTimer`的类，它总是在屏幕完成一次更新之前启动，它的接口设计的和`NSTimer`很类似，所以它实际上就是一个内置实现的替代，但是和`timeInterval`以秒为单位不同，`CADisplayLink`有一个整型的`frameInterval`属性，指定了间隔多少帧之后才执行。默认值是1，意味着每次屏幕更新之前都会执行一次。但是如果动画的代码执行起来超过了六十分之一秒，你可以指定`frameInterval`为2，就是说动画每隔一帧执行一次（一秒钟30帧）或者3，也就是一秒钟20次，等等。

用`CADisplayLink`而不是`NSTimer`，会保证帧率足够连续，使得动画看起来更加平滑，但即使`CADisplayLink`也不能_保证_每一帧都按计划执行，一些失去控制的离散的任务或者事件（例如资源紧张的后台程序）可能会导致动画偶尔地丢帧。当使用`NSTimer`的时候，一旦有机会计时器就会开启，但是`CADisplayLink`却不一样：如果它丢失了帧，就会直接忽略它们，然后在下一次更新的时候接着运行。

### 计算帧的持续时间

无论是使用`NSTimer`还是`CADisplayLink`，我们仍然需要处理一帧的时间超出了预期的六十分之一秒。由于我们不能够计算出一帧真实的持续时间，所以需要手动测量。我们可以在每帧开始刷新的时候用`CACurrentMediaTime()`记录当前时间，然后和上一帧记录的时间去比较。

通过比较这些时间，我们就可以得到真实的每帧持续的时间，然后代替硬编码的六十分之一秒。我们来更新一下上个例子（见清单11.2）。

清单11.2 通过测量没帧持续的时间来使得动画更加平滑

```text
@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *containerView;
@property (nonatomic, strong) UIImageView *ballView;
@property (nonatomic, strong) CADisplayLink *timer;
@property (nonatomic, assign) CFTimeInterval duration;
@property (nonatomic, assign) CFTimeInterval timeOffset;
@property (nonatomic, assign) CFTimeInterval lastStep;
@property (nonatomic, strong) id fromValue;
@property (nonatomic, strong) id toValue;

@end

@implementation ViewController

...

- (void)animate
{
    //reset ball to top of screen
    self.ballView.center = CGPointMake(150, 32);
    //configure the animation
    self.duration = 1.0;
    self.timeOffset = 0.0;
    self.fromValue = [NSValue valueWithCGPoint:CGPointMake(150, 32)];
    self.toValue = [NSValue valueWithCGPoint:CGPointMake(150, 268)];
    //stop the timer if it's already running
    [self.timer invalidate];
    //start the timer
    self.lastStep = CACurrentMediaTime();
    self.timer = [CADisplayLink displayLinkWithTarget:self
                                             selector:@selector(step:)];
    [self.timer addToRunLoop:[NSRunLoop mainRunLoop]
                     forMode:NSDefaultRunLoopMode];
}

- (void)step:(CADisplayLink *)timer
{
    //calculate time delta
    CFTimeInterval thisStep = CACurrentMediaTime();
    CFTimeInterval stepDuration = thisStep - self.lastStep;
    self.lastStep = thisStep;
    //update time offset
    self.timeOffset = MIN(self.timeOffset + stepDuration, self.duration);
    //get normalized time offset (in range 0 - 1)
    float time = self.timeOffset / self.duration;
    //apply easing
    time = bounceEaseOut(time);
    //interpolate position
    id position = [self interpolateFromValue:self.fromValue toValue:self.toValue
                                        time:time];
    //move ball view to new position
    self.ballView.center = [position CGPointValue];
    //stop the timer if we've reached the end of the animation
    if (self.timeOffset >= self.duration) {
        [self.timer invalidate];
        self.timer = nil;
    }
}

@end
```

### Run Loop 模式

注意到当创建`CADisplayLink`的时候，我们需要指定一个`run loop`和`run loop mode`，对于run loop来说，我们就使用了主线程的run loop，因为任何用户界面的更新都需要在主线程执行，但是模式的选择就并不那么清楚了，每个添加到run loop的任务都有一个指定了优先级的模式，为了保证用户界面保持平滑，iOS会提供和用户界面相关任务的优先级，而且当UI很活跃的时候的确会暂停一些别的任务。

一个典型的例子就是当是用`UIScrollview`滑动的时候，重绘滚动视图的内容会比别的任务优先级更高，所以标准的`NSTimer`和网络请求就不会启动，一些常见的run loop模式如下：

* `NSDefaultRunLoopMode` - 标准优先级
* `NSRunLoopCommonModes` - 高优先级
* `UITrackingRunLoopMode` - 用于`UIScrollView`和别的控件的动画

在我们的例子中，我们是用了`NSDefaultRunLoopMode`，但是不能保证动画平滑的运行，所以就可以用`NSRunLoopCommonModes`来替代。但是要小心，因为如果动画在一个高帧率情况下运行，你会发现一些别的类似于定时器的任务或者类似于滑动的其他iOS动画会暂停，直到动画结束。

同样可以同时对`CADisplayLink`指定多个run loop模式，于是我们可以同时加入`NSDefaultRunLoopMode`和`UITrackingRunLoopMode`来保证它不会被滑动打断，也不会被其他UIKit控件动画影响性能，像这样：

```text
self.timer = [CADisplayLink displayLinkWithTarget:self selector:@selector(step:)];
[self.timer addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSDefaultRunLoopMode];
[self.timer addToRunLoop:[NSRunLoop mainRunLoop] forMode:UITrackingRunLoopMode];
```

和`CADisplayLink`类似，`NSTimer`同样也可以使用不同的run loop模式配置，通过别的函数，而不是`+scheduledTimerWithTimeInterval:`构造器

```text
self.timer = [NSTimer timerWithTimeInterval:1/60.0
                                 target:self
                               selector:@selector(step:)
                               userInfo:nil
                                repeats:YES];
[[NSRunLoop mainRunLoop] addTimer:self.timer
                          forMode:NSRunLoopCommonModes];
```

## 物理模拟

即使使用了基于定时器的动画来复制第10章中关键帧的行为，但还是会有一些本质上的区别：在关键帧的实现中，我们提前计算了所有帧，但是在新的解决方案中，我们实际上实在按需要在计算。意义在于我们可以根据用户输入实时修改动画的逻辑，或者和别的实时动画系统例如物理引擎进行整合。

### Chipmunk

我们来基于物理学创建一个真实的重力模拟效果来取代当前基于缓冲的弹性动画，但即使模拟2D的物理效果就已近极其复杂了，所以就不要尝试去实现它了，直接用开源的物理引擎库好了。

我们将要使用的物理引擎叫做Chipmunk。另外的2D物理引擎也同样可以（例如Box2D），但是Chipmunk使用纯C写的，而不是C++，好处在于更容易和Objective-C项目整合。Chipmunk有很多版本，包括一个和Objective-C绑定的“indie”版本。C语言的版本是免费的，所以我们就用它好了。在本书写作的时候6.1.4是最新的版本；你可以从[http://chipmunk-physics.net下载它。](http://chipmunk-physics.net下载它。)

Chipmunk完整的物理引擎相当巨大复杂，但是我们只会使用如下几个类：

* `cpSpace` - 这是所有的物理结构体的容器。它有一个大小和一个可选的重力矢量
* `cpBody` - 它是一个固态无弹力的刚体。它有一个坐标，以及其他物理属性，例如质量，运动和摩擦系数等等。
* `cpShape` - 它是一个抽象的几何形状，用来检测碰撞。可以给结构体添加一个多边形，而且`cpShape`有各种子类来代表不同形状的类型。

在例子中，我们来对一个木箱建模，然后在重力的影响下下落。我们来创建一个`Crate`类，包含屏幕上的可视效果（一个`UIImageView`）和一个物理模型（一个`cpBody`和一个`cpPolyShape`，一个`cpShape`的多边形子类来代表矩形木箱）。

用C版本的Chipmunk会带来一些挑战，因为它现在并不支持Objective-C的引用计数模型，所以我们需要准确的创建和释放对象。为了简化，我们把`cpShape`和`cpBody`的生命周期和`Crate`类进行绑定，然后在木箱的`-init`方法中创建，在`-dealloc`中释放。木箱物理属性的配置很复杂，所以阅读了Chipmunk文档会很有意义。

视图控制器用来管理`cpSpace`，还有和之前一样的计时器逻辑。在每一步中，我们更新`cpSpace`（用来进行物理计算和所有结构体的重新摆放）然后迭代对象，然后再更新我们的木箱视图的位置来匹配木箱的模型（在这里，实际上只有一个结构体，但是之后我们将要添加更多）。

Chipmunk使用了一个和UIKit颠倒的坐标系（Y轴向上为正方向）。为了使得物理模型和视图之间的同步更简单，我们需要通过使用`geometryFlipped`属性翻转容器视图的集合坐标（第3章中有提到），于是模型和视图都共享一个相同的坐标系。

具体的代码见清单11.3。注意到我们并没有在任何地方释放`cpSpace`对象。在这个例子中，内存空间将会在整个app的生命周期中一直存在，所以这没有问题。但是在现实世界的场景中，我们需要像创建木箱结构体和形状一样去管理我们的空间，封装在标准的Cocoa对象中，然后来管理Chipmunk对象的生命周期。图11.1展示了掉落的木箱。

清单11.3 使用物理学来对掉落的木箱建模

```text
#import "ViewController.h" 
#import <QuartzCore/QuartzCore.h>
#import "chipmunk.h"

@interface Crate : UIImageView

@property (nonatomic, assign) cpBody *body;
@property (nonatomic, assign) cpShape *shape;

@end

@implementation Crate

#define MASS 100

- (id)initWithFrame:(CGRect)frame
{
    if ((self = [super initWithFrame:frame])) {
        //set image
        self.image = [UIImage imageNamed:@"Crate.png"];
        self.contentMode = UIViewContentModeScaleAspectFill;
        //create the body
        self.body = cpBodyNew(MASS, cpMomentForBox(MASS, frame.size.width, frame.size.height));
        //create the shape
        cpVect corners[] = {
            cpv(0, 0),
            cpv(0, frame.size.height),
            cpv(frame.size.width, frame.size.height),
            cpv(frame.size.width, 0),
        };
        self.shape = cpPolyShapeNew(self.body, 4, corners, cpv(-frame.size.width/2, -frame.size.height/2));
        //set shape friction & elasticity
        cpShapeSetFriction(self.shape, 0.5);
        cpShapeSetElasticity(self.shape, 0.8);
        //link the crate to the shape
        //so we can refer to crate from callback later on
        self.shape->data = (__bridge void *)self;
        //set the body position to match view
        cpBodySetPos(self.body, cpv(frame.origin.x + frame.size.width/2, 300 - frame.origin.y - frame.size.height/2));
    }
    return self;
}

- (void)dealloc
{
    //release shape and body
    cpShapeFree(_shape);
    cpBodyFree(_body);
}

@end

@interface ViewController ()

@property (nonatomic, weak) IBOutlet UIView *containerView;
@property (nonatomic, assign) cpSpace *space;
@property (nonatomic, strong) CADisplayLink *timer;
@property (nonatomic, assign) CFTimeInterval lastStep;

@end

@implementation ViewController

#define GRAVITY 1000

- (void)viewDidLoad
{
    //invert view coordinate system to match physics
    self.containerView.layer.geometryFlipped = YES;
    //set up physics space
    self.space = cpSpaceNew();
    cpSpaceSetGravity(self.space, cpv(0, -GRAVITY));
    //add a crate
    Crate *crate = [[Crate alloc] initWithFrame:CGRectMake(100, 0, 100, 100)];
    [self.containerView addSubview:crate];
    cpSpaceAddBody(self.space, crate.body);
    cpSpaceAddShape(self.space, crate.shape);
    //start the timer
    self.lastStep = CACurrentMediaTime();
    self.timer = [CADisplayLink displayLinkWithTarget:self
                                             selector:@selector(step:)];
    [self.timer addToRunLoop:[NSRunLoop mainRunLoop]
                     forMode:NSDefaultRunLoopMode];
}

void updateShape(cpShape *shape, void *unused)
{
    //get the crate object associated with the shape
    Crate *crate = (__bridge Crate *)shape->data;
    //update crate view position and angle to match physics shape
    cpBody *body = shape->body;
    crate.center = cpBodyGetPos(body);
    crate.transform = CGAffineTransformMakeRotation(cpBodyGetAngle(body));
}

- (void)step:(CADisplayLink *)timer
{
    //calculate step duration
    CFTimeInterval thisStep = CACurrentMediaTime();
    CFTimeInterval stepDuration = thisStep - self.lastStep;
    self.lastStep = thisStep;
    //update physics
    cpSpaceStep(self.space, stepDuration);
    //update all the shapes
    cpSpaceEachShape(self.space, &updateShape, NULL);
}

@end
```

![&#x56FE;11.1](.gitbook/assets/11.1.jpeg)

图11.1 一个木箱图片，根据模拟的重力掉落

### 添加用户交互

下一步就是在视图周围添加一道不可见的墙，这样木箱就不会掉落出屏幕之外。或许你会用另一个矩形的`cpPolyShape`来实现，就和之前创建木箱那样，但是我们需要检测的是木箱何时离开视图，而不是何时碰撞，所以我们需要一个空心而不是固体矩形。

我们可以通过给`cpSpace`添加四个`cpSegmentShape`对象（`cpSegmentShape`代表一条直线，所以四个拼起来就是一个矩形）。然后赋给空间的`staticBody`属性（一个不被重力影响的结构体）而不是像木箱那样一个新的`cpBody`实例，因为我们不想让这个边框矩形滑出屏幕或者被一个下落的木箱击中而消失。

同样可以再添加一些木箱来做一些交互。最后再添加一个加速器，这样可以通过倾斜手机来调整重力矢量（为了测试需要在一台真实的设备上运行程序，因为模拟器不支持加速器事件，即使旋转屏幕）。清单11.4展示了更新后的代码，运行结果见图11.2。

由于示例只支持横屏模式，所以交换加速计矢量的x和y值。如果在竖屏下运行程序，请把他们换回来，不然重力方向就错乱了。试一下就知道了，木箱会沿着横向移动。

清单11.4 使用围墙和多个木箱的更新后的代码

```text
- (void)addCrateWithFrame:(CGRect)frame
{
    Crate *crate = [[Crate alloc] initWithFrame:frame];
    [self.containerView addSubview:crate];
    cpSpaceAddBody(self.space, crate.body);
    cpSpaceAddShape(self.space, crate.shape);
}

- (void)addWallShapeWithStart:(cpVect)start end:(cpVect)end
{
    cpShape *wall = cpSegmentShapeNew(self.space->staticBody, start, end, 1);
    cpShapeSetCollisionType(wall, 2);
    cpShapeSetFriction(wall, 0.5);
    cpShapeSetElasticity(wall, 0.8);
    cpSpaceAddStaticShape(self.space, wall);
}

- (void)viewDidLoad
{
    //invert view coordinate system to match physics
    self.containerView.layer.geometryFlipped = YES;
    //set up physics space
    self.space = cpSpaceNew();
    cpSpaceSetGravity(self.space, cpv(0, -GRAVITY));
    //add wall around edge of view
    [self addWallShapeWithStart:cpv(0, 0) end:cpv(300, 0)];
    [self addWallShapeWithStart:cpv(300, 0) end:cpv(300, 300)];
    [self addWallShapeWithStart:cpv(300, 300) end:cpv(0, 300)];
    [self addWallShapeWithStart:cpv(0, 300) end:cpv(0, 0)];
    //add a crates
    [self addCrateWithFrame:CGRectMake(0, 0, 32, 32)];
    [self addCrateWithFrame:CGRectMake(32, 0, 32, 32)];
    [self addCrateWithFrame:CGRectMake(64, 0, 64, 64)];
    [self addCrateWithFrame:CGRectMake(128, 0, 32, 32)];
    [self addCrateWithFrame:CGRectMake(0, 32, 64, 64)];
    //start the timer
    self.lastStep = CACurrentMediaTime();
    self.timer = [CADisplayLink displayLinkWithTarget:self
                                             selector:@selector(step:)];
    [self.timer addToRunLoop:[NSRunLoop mainRunLoop]
                     forMode:NSDefaultRunLoopMode];
    //update gravity using accelerometer
    [UIAccelerometer sharedAccelerometer].delegate = self;
    [UIAccelerometer sharedAccelerometer].updateInterval = 1/60.0;
}

- (void)accelerometer:(UIAccelerometer *)accelerometer didAccelerate:(UIAcceleration *)acceleration
{
    //update gravity
    cpSpaceSetGravity(self.space, cpv(acceleration.y * GRAVITY, -acceleration.x * GRAVITY));
}
```

![&#x56FE;11.2](.gitbook/assets/11.2.jpeg)

图11.1 真实引力场下的木箱交互

### 模拟时间以及固定的时间步长

对于实现动画的缓冲效果来说，计算每帧持续的时间是一个很好的解决方案，但是对模拟物理效果并不理想。通过一个可变的时间步长来实现有着两个弊端：

* 如果时间步长不是固定的，精确的值，物理效果的模拟也就随之不确定。这意味着即使是传入相同的输入值，也可能在不同场合下有着不同的效果。有时候没多大影响，但是在基于物理引擎的游戏下，玩家就会由于相同的操作行为导致不同的结果而感到困惑。同样也会让测试变得麻烦。
* 由于性能故常造成的丢帧或者像电话呼入的中断都可能会造成不正确的结果。考虑一个像子弹那样快速移动物体，每一帧的更新都需要移动子弹，检测碰撞。如果两帧之间的时间加长了，子弹就会在这一步移动更远的距离，穿过围墙或者是别的障碍，这样就丢失了碰撞。

我们想得到的理想的效果就是通过固定的时间步长来计算物理效果，但是在屏幕发生重绘的时候仍然能够同步更新视图（可能会由于在我们控制范围之外造成不可预知的效果）。

幸运的是，由于我们的模型（在这个例子中就是Chipmunk的`cpSpace`中的`cpBody`）被视图（就是屏幕上代表木箱的`UIView`对象）分离，于是就很简单了。我们只需要根据屏幕刷新的时间跟踪时间步长，然后根据每帧去计算一个或者多个模拟出来的效果。

我们可以通过一个简单的循环来实现。通过每次`CADisplayLink`的启动来通知屏幕将要刷新，然后记录下当前的`CACurrentMediaTime()`。我们需要在一个小增量中提前重复物理模拟（这里用120分之一秒）直到赶上显示的时间。然后更新我们的视图，在屏幕刷新的时候匹配当前物理结构体的显示位置。

清单11.5展示了固定时间步长版本的代码

清单11.5 固定时间步长的木箱模拟

```text
#define SIMULATION_STEP (1/120.0)

- (void)step:(CADisplayLink *)timer
{
    //calculate frame step duration
    CFTimeInterval frameTime = CACurrentMediaTime();
    //update simulation
    while (self.lastStep < frameTime) {
        cpSpaceStep(self.space, SIMULATION_STEP);
        self.lastStep += SIMULATION_STEP;
    }
    ￼
    //update all the shapes
    cpSpaceEachShape(self.space, &updateShape, NULL);
}
```

### 避免死亡螺旋

当使用固定的模拟时间步长时候，有一件事情一定要注意，就是用来计算物理效果的现实世界的时间并不会加速模拟时间步长。在我们的例子中，我们随意选择了120分之一秒来模拟物理效果。Chipmunk很快，我们的例子也很简单，所以`cpSpaceStep()`会完成的很好，不会延迟帧的更新。

但是如果场景很复杂，比如有上百个物体之间的交互，物理计算就会很复杂，`cpSpaceStep()`的计算也可能会超出1/120秒。我们没有测量出物理步长的时间，因为我们假设了相对于帧刷新来说并不重要，但是如果模拟步长更久的话，就会延迟帧率。

如果帧刷新的时间延迟的话会变得很糟糕，我们的模拟需要执行更多的次数来同步真实的时间。这些额外的步骤就会继续延迟帧的更新，等等。这就是所谓的死亡螺旋，因为最后的结果就是帧率变得越来越慢，直到最后应用程序卡死了。

我们可以通过添加一些代码在设备上来对物理步骤计算真实世界的时间，然后自动调整固定时间步长，但是实际上它不可行。其实只要保证你给容错留下足够的边长，然后在期望支持的最慢的设备上进行测试就可以了。如果物理计算超过了模拟时间的50%，就需要考虑增加模拟时间步长（或者简化场景）。如果模拟时间步长增加到超过1/60秒（一个完整的屏幕更新时间），你就需要减少动画帧率到一秒30帧或者增加`CADisplayLink`的`frameInterval`来保证不会随机丢帧，不然你的动画将会看起来不平滑。

## 总结

在这一章中，我们了解了如何通过一个计时器创建一帧帧的实时动画，包括缓冲，物理模拟等等一系列动画技术，以及用户输入（通过加速计）。

在第三部分中，我们将研究动画性能是如何被被设备限制所影响的，以及如何调整我们的代码来活的足够好的帧率。

