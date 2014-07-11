---
layout: post
categories: [iOS]
---
#iOS RunLoop简介
第一次看到RunLoop应该是在在用于“阻塞”主线程的用法中。
```C
// 主线程
while (!doSthFinished)
{
    [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
}

// 其他线程
...
// do sth
dispatch_async(dispatch_get_main_queue(), ^{
    _doSthFinished = YES;
});
```
这样的代码看上去让主线程暂停在了while循环中，令人感到神奇的是，触摸事件却能正常触发，为了能明白其中的道理，先从最简单一个Demo开始吧。
```C
- (void)startThread
{
    NSThread* thread = [[NSThread alloc] initWithTarget:self selector:@selector(threadMain) object:nil];
    [thread start];
}

- (void)threadMain
{
    [NSTimer scheduledTimerWithTimeInterval:1 target:self selector:@selector(onTimer) userInfo:nil repeats:NO];
}

- (void)onTimer
{
    NSLog(@"enter:%s", __func__);
}
```
这段代码很简单，启动一个线程，线程函数中启动一个1秒的timer，timer触发时打印一下日志。
运行的结果是onTimer并没有被调用。
这里引申出了一个概念，timer是和所在线程相关的，这里的线程在threadMain函数结束后也随之结束了，所以timer没有触发。
下面进行另外一个实验。
```C
- (void)threadMain
{
    [self performSelector:@selector(someMethod) withObject:nil afterDelay:1];
}

- (void)someMethod
{
    NSLog(@"enter:%s", __func__);
}
```
和timer一样，someMethod函数也没有调用。
可是在主线程这么用是没问题的啊。
想办法让这样的代码正常工作吧。
```C
- (void)threadMain
{
    [self performSelector:@selector(someMethod) withObject:nil afterDelay:1];
    [NSTimer scheduledTimerWithTimeInterval:2 target:self selector:@selector(onTimer) userInfo:nil repeats:NO];
    [[NSRunLoop currentRunLoop] run];
    NSLog(@"leave:%s", __func__);
}

- (void)onTimer
{
    NSLog(@"enter:%s", __func__);
}

- (void)someMethod
{
    NSLog(@"enter:%s", __func__);
}
日志输出:
2014-07-11 14:00:09.132 RunLoopTest[2666:370b] enter:-[ViewController someMethod]
2014-07-11 14:00:10.133 RunLoopTest[2666:370b] enter:-[ViewController onTimer]
2014-07-11 14:00:10.135 RunLoopTest[2666:370b] leave:-[ViewController threadMain]
```
从日志输出看出，加上
```C
[[NSRunLoop currentRunLoop] run];
```
这句，代码就工作正常了。
那么runloop的run又做了点什么事件呢，先看看文档。
```C
- (void)run
Description 
Puts the receiver into a permanent loop, during which time it processes data from all attached input sources.
If no input sources or timers are attached to the run loop, this method exits immediately; otherwise, it runs the receiver in the NSDefaultRunLoopMode by repeatedly invoking runMode:beforeDate:. In other words, this method effectively begins an infinite loop that processes data from the run loop’s input sources and timers.
```
大概意思是将runloop置于一个循环之中，直到和它关联的input sources都处理完毕。如果调用该方法时候没有关联的input sources，那么该函数会立即返回。
这里简单的将timer和performSelector都看成input sources了。
其他平台也有类似和消息分发吧。
文档中还说到mode概念，其中提到了`NSDefaultRunLoopMode`，这mode又是怎么一回事呢，先对上面的代码做点小修改。
```
- (void)threadMain
{
    [self performSelector:@selector(someMethod) withObject:nil afterDelay:1];
    
    NSTimer* timer = [NSTimer timerWithTimeInterval:2 target:self selector:@selector(onTimer) userInfo:nil repeats:NO];
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:@"someMode"];

    [[NSRunLoop currentRunLoop] run];
    
    NSLog(@"leave:%s", __func__);
}

- (void)onTimer
{
    NSLog(@"enter:%s", __func__);
}

- (void)someMethod
{
    NSLog(@"enter:%s", __func__);
}
日志输出：
2014-07-11 14:25:01.201 RunLoopTest[2685:370b] enter:-[ViewController someMethod]
2014-07-11 14:25:01.205 RunLoopTest[2685:370b] leave:-[ViewController threadMain]
```
这里很明显的看到onTimer没有被调用。
按照run函数的文档，调用run是将runloop以NSDefaultRunLoopMode模式运行，而我们这里将timer加入的是@"someMode"，runloop在NSDefaultRunLoopMode模式下不会处理其他模式下的事件。为了证实这一点，我们将runloop运行到@"someMode"看看结果。
```
[[NSRunLoop currentRunLoop] runMode:@"someMode" beforeDate:[NSDate distantFuture]];
日志输出：
2014-07-11 14:33:39.561 RunLoopTest[2691:370b] enter:-[ViewController onTimer]
2014-07-11 14:33:39.565 RunLoopTest[2691:370b] leave:-[ViewController threadMain]
```
证实了上面的结论。
系统定义的模式，除了NSDefaultRunLoopMode，还有其他模式么。
在头文件里找到了NSRunLoopCommonModes，这个又是什么模式呢。
```
This is a configurable group of commonly used modes. Associating an input source with this mode also associates it with each of the modes in the group. For Cocoa applications, this set includes the default, modal, and event tracking modes by default. Core Foundation includes just the default mode initially. You can add custom modes to the set using the CFRunLoopAddCommonMode function.
来源：
https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW1
```
简单来说NSRunLoopCommonModes是一个mode合集，在Cocoa应用程序中包含 default, modal,  event tracking 三种模式，关于这几种模式的说明，参见来源网页。
如果一个input source是加入到NSRunLoopCommonModes的话，那么在上面三种模式都会被触发。
runloop可以这样么:
```
[[NSRunLoop currentRunLoop] runMode:NSRunLoopCommonModes beforeDate:[NSDate distantFuture]];
```
答案是不行，runloop只能在一种模式下运行，而NSRunLoopCommonModes不是一种模式，是一组模式。
比如说你是runloop，你女朋友是input source，你女朋友可以叫你在家的，在公司，在路上都要给她打电话，而你本身不能同时又在家，又在公司，又在路上。

文章够长了，下回再说。

