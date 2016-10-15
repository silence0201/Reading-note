##定时帧

动画看起来是用来显示一段连续的运动过程，但实际上当在固定位置上展示像素的时候并不能做到这一点。一般来说这种显示都无法做到连续的移动，能做的仅仅是足够快地展示一系列静态图片，只是看起来像是做了运动。

我们之前提到过iOS按照每秒60次刷新屏幕，然后`CAAnimation`计算出需要展示的新的帧，然后在每次屏幕更新的时候同步绘制上去，`CAAnimation`最机智的地方在于每次刷新需要展示的时候去计算插值和缓冲。

在第10章中，我们解决了如何自定义缓冲函数，然后根据需要展示的帧的数组来告诉`CAKeyframeAnimation`的实例如何去绘制。所有的Core Animation实际上都是按照一定的序列来显示这些帧，那么我们可以自己做到这些么？

###`NSTimer`

实际上，我们在第三章“图层几何学”中已经做过类似的东西，就是时钟那个例子，我们用了`NSTimer`来对钟表的指针做定时动画，一秒钟更新一次，但是如果我们把频率调整成一秒钟更新60次的话，原理是完全相同的。

我们来试着用`NSTimer`来修改第十章中弹性球的例子。由于现在我们在定时器启动之后连续计算动画帧，我们需要在类中添加一些额外的属性来存储动画的`fromValue`，`toValue`，`duration`和当前的`timeOffset`（见清单11.1）。

清单11.1 使用`NSTimer`实现弹性球动画

```objective-c
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

###`CADisplayLink`

`CADisplayLink`是CoreAnimation提供的另一个类似于`NSTimer`的类，它总是在屏幕完成一次更新之前启动，它的接口设计的和`NSTimer`很类似，所以它实际上就是一个内置实现的替代，但是和`timeInterval`以秒为单位不同，`CADisplayLink`有一个整型的`frameInterval`属性，指定了间隔多少帧之后才执行。默认值是1，意味着每次屏幕更新之前都会执行一次。但是如果动画的代码执行起来超过了六十分之一秒，你可以指定`frameInterval`为2，就是说动画每隔一帧执行一次（一秒钟30帧）或者3，也就是一秒钟20次，等等。

用`CADisplayLink`而不是`NSTimer`，会保证帧率足够连续，使得动画看起来更加平滑，但即使`CADisplayLink`也不能*保证*每一帧都按计划执行，一些失去控制的离散的任务或者事件（例如资源紧张的后台程序）可能会导致动画偶尔地丢帧。当使用`NSTimer`的时候，一旦有机会计时器就会开启，但是`CADisplayLink`却不一样：如果它丢失了帧，就会直接忽略它们，然后在下一次更新的时候接着运行。

###计算帧的持续时间

无论是使用`NSTimer`还是`CADisplayLink`，我们仍然需要处理一帧的时间超出了预期的六十分之一秒。由于我们不能够计算出一帧真实的持续时间，所以需要手动测量。我们可以在每帧开始刷新的时候用`CACurrentMediaTime()`记录当前时间，然后和上一帧记录的时间去比较。

通过比较这些时间，我们就可以得到真实的每帧持续的时间，然后代替硬编码的六十分之一秒。我们来更新一下上个例子（见清单11.2）。

清单11.2 通过测量没帧持续的时间来使得动画更加平滑

```objective-c
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

###Run Loop 模式

注意到当创建`CADisplayLink`的时候，我们需要指定一个`run loop`和`run loop mode`，对于run loop来说，我们就使用了主线程的run loop，因为任何用户界面的更新都需要在主线程执行，但是模式的选择就并不那么清楚了，每个添加到run loop的任务都有一个指定了优先级的模式，为了保证用户界面保持平滑，iOS会提供和用户界面相关任务的优先级，而且当UI很活跃的时候的确会暂停一些别的任务。

一个典型的例子就是当是用`UIScrollview`滑动的时候，重绘滚动视图的内容会比别的任务优先级更高，所以标准的`NSTimer`和网络请求就不会启动，一些常见的run loop模式如下：

* `NSDefaultRunLoopMode` - 标准优先级
* `NSRunLoopCommonModes` - 高优先级
* `UITrackingRunLoopMode` - 用于`UIScrollView`和别的控件的动画

在我们的例子中，我们是用了`NSDefaultRunLoopMode`，但是不能保证动画平滑的运行，所以就可以用`NSRunLoopCommonModes`来替代。但是要小心，因为如果动画在一个高帧率情况下运行，你会发现一些别的类似于定时器的任务或者类似于滑动的其他iOS动画会暂停，直到动画结束。

同样可以同时对`CADisplayLink`指定多个run loop模式，于是我们可以同时加入`NSDefaultRunLoopMode`和`UITrackingRunLoopMode`来保证它不会被滑动打断，也不会被其他UIKit控件动画影响性能，像这样：

    self.timer = [CADisplayLink displayLinkWithTarget:self selector:@selector(step:)];
    [self.timer addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSDefaultRunLoopMode];
    [self.timer addToRunLoop:[NSRunLoop mainRunLoop] forMode:UITrackingRunLoopMode];

和`CADisplayLink`类似，`NSTimer`同样也可以使用不同的run loop模式配置，通过别的函数，而不是`+scheduledTimerWithTimeInterval:`构造器

    self.timer = [NSTimer timerWithTimeInterval:1/60.0
                                     target:self
                                   selector:@selector(step:)
                                   userInfo:nil
                                    repeats:YES];
    [[NSRunLoop mainRunLoop] addTimer:self.timer
                              forMode:NSRunLoopCommonModes];
                              