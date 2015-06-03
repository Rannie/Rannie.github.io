---
layout: post
title:  "从 Objective-C 对象到 Runtime （RunLoop）"
date:   2015-06-03 09:31:00
categories: iOS

---

RunLoop 是 iOS/Mac 开发中一个很基础的概念，虽然大部分时间在代码中跟它很少打交道，不过对于一个开发人员还是需要了解一些相关的机制和知识。

###为何要有 RunLoop


一般来说，一个线程一次只能执行一个任务，执行完成就会退出，我们程序的运行如果没有事件需要处理时也不能退出，所以大概就需要下面这样一个机制，让线程能处理事件但是不退出：


	function loop() {
	    initialize();
	    do {
	        var message = get_next_message();
	        process_message(message);
	    } while (message != quit);
	}

这种模型通常被称作 Event Loop。 Event Loop 在很多系统和框架里都有实现，比如 Node.js 的事件处理，比如 Windows 程序的消息循环，再比如 OSX/iOS 里的 RunLoop。实现这种模型的关键点在于：如何管理事件/消息，如何让线程在没有处理消息时休眠以避免资源占用、在有消息到来时立刻被唤醒。

所以，RunLoop 实际上就是一个对象，这个对象管理了其需要处理的事件和消息，并提供了一个入口函数来执行上面 Event Loop 的逻辑。线程执行了这个函数后，就会一直处于这个函数内部 "接受消息->等待->处理" 的循环中，直到这个循环结束（比如传入 quit 的消息），函数返回。

总结来说:

* 使程序一直运行并接受用户的输入
* 决定在何时去处理哪些 Event
* 调用解耦 
* 节约 CPU 时间

###RunLoop in Cocoa

OSX/iOS 系统中，提供了两个这样的对象：NSRunLoop 和 CFRunLoopRef。
CFRunLoopRef 是在 CoreFoundation 框架内的，它提供了纯 C 函数的 API，所有这些 API 都是线程安全的。
NSRunLoop 是基于 CFRunLoopRef 的封装，提供了面向对象的 API，但是这些 API 不是线程安全的。

RunLoop 涉及到的一些常见的类或者方法等：

NSTimer, UIEvent, Autorelease, <br>
NSObject 的一些方法如 *performSelector:afterDelay:....* <br>
CADisplayLink, CATransition, CAAnimation <br>
dispatch_get_main_queue() <br>
NSURLConnection <AFNetworking>

RunLoop 处理的事件源通常有大概如下几种：

observer, block, dispatch_main_queue, timer, source0, source1.

###RunLoop 与线程的关系

线程和 RunLoop 之间是一一对应的，其关系是保存在一个全局的 Dictionary 里。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 RunLoop（主线程除外）。

###RunLoop 成员类们

在 CoreFoundation 里面关于 RunLoop 有5个类:

* CFRunLoopRef
* CFRunLoopModeRef
* CFRunLoopSourceRef
* CFRunLoopTimerRef
* CFRunLoopObserverRef

我们刚才提到过 CFRunLoopRef 与 thread 是一一对应的，然后一个 RunLoop 又包含若干个 Mode，每个 Mode 又包含若干个 Source/Timer/Observer。每次调用 RunLoop 的主函数时，只能指定其中一个 Mode，这个Mode被称作 CurrentMode。如果需要切换 Mode，只能退出 Loop，再重新指定一个 Mode 进入。这样做主要是为了分隔开不同组的 Source/Timer/Observer，让其互不影响。

#####CFRunLoopSourceRef

CFRunLoopSourceRef 是事件产生的地方。

有两个版本的 Source ：

* Source0: App 内部事件回调，如 UIEvent, CFSocket 等。 Source0 只包含了一个回调（函数指针），它并不能主动触发事件。使用时，你需要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop，让其处理这个事件。
* Source1: 由 RunLoop 和内核管理， mach port 驱动，如 CFMachPort, CFMessagePort。 Source1 包含了一个 mach_port 和一个回调（函数指针），被用于通过内核和其他线程相互发送消息。这种 Source 能主动唤醒 RunLoop 的线程。

#####CFRunLoopObserverRef

CFRunLoopObserverRef 是观察者，每个 Observer 都包含了一个回调（函数指针），当 RunLoop 的状态发生变化时，观察者就能通过回调接受到这个变化。可以观测的时间点官方的枚举给出了以下几个：

	typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
	    kCFRunLoopEntry         = (1UL << 0), // 即将进入Loop
	    kCFRunLoopBeforeTimers  = (1UL << 1), // 即将处理 Timer
	    kCFRunLoopBeforeSources = (1UL << 2), // 即将处理 Source
	    kCFRunLoopBeforeWaiting = (1UL << 5), // 即将进入休眠
	    kCFRunLoopAfterWaiting  = (1UL << 6), // 刚从休眠中唤醒
	    kCFRunLoopExit          = (1UL << 7), // 即将退出Loop
	};
	
上面的 Source/Timer/Observer 被统称为 mode item，一个 item 可以被同时加入多个 mode。但一个 item 被重复加入同一个 mode 时是不会有效果的。如果一个 mode 中一个 item 都没有，则 RunLoop 会直接退出，不进入循环。

#####RunLoop Mode

CFRunLoopMode 和 CFRunLoop 的结构大致如下：

	struct __CFRunLoopMode {
	    CFStringRef _name;            // Mode Name, 例如 @"kCFRunLoopDefaultMode"
	    CFMutableSetRef _sources0;    // Set<CFRunLoopSourceRef>
	    CFMutableSetRef _sources1;    // Set<CFRunLoopSourceRef>
	    CFMutableArrayRef _observers; // Array<CFRunLoopObserverRef>
	    CFMutableArrayRef _timers;    // Array<CFRunLoopTimerRef>
	    ...
	};
	 
	struct __CFRunLoop {
	    CFMutableSetRef _commonModes;     // Set<CFStringRef>
	    CFMutableSetRef _commonModeItems; // Set<Source/Observer/Timer>
	    CFRunLoopModeRef _currentMode;    // Current Runloop Mode
	    CFMutableSetRef _modes;           // Set<CFRunLoopModeRef>
	    ...
	};

这里有个概念叫 "CommonModes"：一个 Mode 可以将自己标记为"Common"属性（通过将其 ModeName 添加到 RunLoop 的 "commonModes" 中）。每当 RunLoop 的内容发生变化时，RunLoop 都会自动将 _commonModeItems 里的 Source/Observer/Timer 同步到具有 "Common" 标记的所有Mode里。

常见的 Mode:

1. kCFRunLoopDefaultMode: App的默认 Mode，通常主线程是在这个 Mode 下运行的。
2. UITrackingRunLoopMode: 界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响。
3. UIInitializationRunLoopMode: 在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用。
4. GSEventReceiveRunLoopMode: 接受系统事件的内部 Mode，通常用不到。
5. kCFRunLoopCommonModes: 这是一个占位的 Mode，没有实际作用。

应用场景举例：主线程的 RunLoop 里有两个预置的 Mode：kCFRunLoopDefaultMode 和 UITrackingRunLoopMode。这两个 Mode 都已经被标记为"Common"属性。DefaultMode 是 App 平时所处的状态，TrackingRunLoopMode 是追踪 ScrollView 滑动时的状态。当你创建一个 Timer 并加到 DefaultMode 时，Timer 会得到重复回调，但此时滑动一个TableView时，RunLoop 会将 mode 切换为 TrackingRunLoopMode，这时 Timer 就不会被回调，并且也不会影响到滑动操作。

有时你需要一个 Timer，在两个 Mode 中都能得到回调，一种办法就是将这个 Timer 分别加入这两个 Mode。还有一种方式，就是将 Timer 加入到顶层的 RunLoop 的 "commonModeItems" 中。"commonModeItems" 被 RunLoop 自动更新到所有具有"Common"属性的 Mode 里去。

	[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
	
###RunLoop 迭代执行顺序

这里有张图清晰的标明了 RunLoop 从开启到执行任务到休眠到唤醒到退出的声明周期。

<img src=http://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_1.png width=80%>

由于 CoreFoundation 开源了 RunLoop 的源代码,整理下大概如下所标示:

	/// 用DefaultMode启动
	void CFRunLoopRun(void) {
	    CFRunLoopRunSpecific(CFRunLoopGetCurrent(), kCFRunLoopDefaultMode, 1.0e10, false);
	}
	 
	/// 用指定的Mode启动，允许设置RunLoop超时时间
	int CFRunLoopRunInMode(CFStringRef modeName, CFTimeInterval seconds, Boolean stopAfterHandle) {
	    return CFRunLoopRunSpecific(CFRunLoopGetCurrent(), modeName, seconds, returnAfterSourceHandled);
	}
	 
	/// RunLoop的实现
	int CFRunLoopRunSpecific(runloop, modeName, seconds, stopAfterHandle) {
	    
	    /// 首先根据modeName找到对应mode
	    CFRunLoopModeRef currentMode = __CFRunLoopFindMode(runloop, modeName, false);
	    /// 如果mode里没有source/timer/observer, 直接返回。
	    if (__CFRunLoopModeIsEmpty(currentMode)) return;
	    
	    /// 1. 通知 Observers: RunLoop 即将进入 loop。
	    __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopEntry);
	    
	    /// 内部函数，进入loop
	    __CFRunLoopRun(runloop, currentMode, seconds, returnAfterSourceHandled) {
	        
	        Boolean sourceHandledThisLoop = NO;
	        int retVal = 0;
	        do {
	 
	            /// 2. 通知 Observers: RunLoop 即将触发 Timer 回调。
	            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeTimers);
	            /// 3. 通知 Observers: RunLoop 即将触发 Source0 (非port) 回调。
	            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeSources);
	            /// 执行被加入的block
	            __CFRunLoopDoBlocks(runloop, currentMode);
	            
	            /// 4. RunLoop 触发 Source0 (非port) 回调。
	            sourceHandledThisLoop = __CFRunLoopDoSources0(runloop, currentMode, stopAfterHandle);
	            /// 执行被加入的block
	            __CFRunLoopDoBlocks(runloop, currentMode);
	 
	            /// 5. 如果有 Source1 (基于port) 处于 ready 状态，直接处理这个 Source1 然后跳转去处理消息。
	            if (__Source0DidDispatchPortLastTime) {
	                Boolean hasMsg = __CFRunLoopServiceMachPort(dispatchPort, &msg)
	                if (hasMsg) goto handle_msg;
	            }
	            
	            /// 通知 Observers: RunLoop 的线程即将进入休眠(sleep)。
	            if (!sourceHandledThisLoop) {
	                __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);
	            }
	            
	            /// 7. 调用 mach_msg 等待接受 mach_port 的消息。线程将进入休眠, 直到被下面某一个事件唤醒。
	            /// • 一个基于 port 的Source 的事件。
	            /// • 一个 Timer 到时间了
	            /// • RunLoop 自身的超时时间到了
	            /// • 被其他什么调用者手动唤醒
	            __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort) {
	                mach_msg(msg, MACH_RCV_MSG, port); // thread wait for receive msg
	            }
	 
	            /// 8. 通知 Observers: RunLoop 的线程刚刚被唤醒了。
	            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopAfterWaiting);
	            
	            /// 收到消息，处理消息。
	            handle_msg:
	 
	            /// 9.1 如果一个 Timer 到时间了，触发这个Timer的回调。
	            if (msg_is_timer) {
	                __CFRunLoopDoTimers(runloop, currentMode, mach_absolute_time())
	            } 
	 
	            /// 9.2 如果有dispatch到main_queue的block，执行block。
	            else if (msg_is_dispatch) {
	                __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
	            } 
	 
	            /// 9.3 如果一个 Source1 (基于port) 发出事件了，处理这个事件
	            else {
	                CFRunLoopSourceRef source1 = __CFRunLoopModeFindSourceForMachPort(runloop, currentMode, livePort);
	                sourceHandledThisLoop = __CFRunLoopDoSource1(runloop, currentMode, source1, msg);
	                if (sourceHandledThisLoop) {
	                    mach_msg(reply, MACH_SEND_MSG, reply);
	                }
	            }
	            
	            /// 执行加入到Loop的block
	            __CFRunLoopDoBlocks(runloop, currentMode);
	            
	 
	            if (sourceHandledThisLoop && stopAfterHandle) {
	                /// 进入loop时参数说处理完事件就返回。
	                retVal = kCFRunLoopRunHandledSource;
	            } else if (timeout) {
	                /// 超出传入参数标记的超时时间了
	                retVal = kCFRunLoopRunTimedOut;
	            } else if (__CFRunLoopIsStopped(runloop)) {
	                /// 被外部调用者强制停止了
	                retVal = kCFRunLoopRunStopped;
	            } else if (__CFRunLoopModeIsEmpty(runloop, currentMode)) {
	                /// source/timer/observer一个都没有了
	                retVal = kCFRunLoopRunFinished;
	            }
	            
	            /// 如果没超时，mode里没空，loop也没被停止，那继续loop。
	        } while (retVal == 0);
	    }
	    
	    /// 10. 通知 Observers: RunLoop 即将退出。
	    __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
	}

当 RunLoop 进行回调时，一般都是通过一个很长的函数调用出去 (call out), 当你在你的代码中下断点调试时，通常能在调用栈上看到这些函数。下面是这几个函数的整理版本，如果你在调用栈中看到这些长函数名，在这里查找一下就能定位到具体的调用地点了：

	{
	    /// 1. 通知Observers，即将进入RunLoop
	    /// 此处有Observer会创建AutoreleasePool: _objc_autoreleasePoolPush();
	    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopEntry);
	    do {
	 
	        /// 2. 通知 Observers: 即将触发 Timer 回调。
	        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeTimers);
	        /// 3. 通知 Observers: 即将触发 Source (非基于port的,Source0) 回调。
	        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeSources);
	        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
	 
	        /// 4. 触发 Source0 (非基于port的) 回调。
	        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__(source0);
	        __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK__(block);
	 
	        /// 6. 通知Observers，即将进入休眠
	        /// 此处有Observer释放并新建AutoreleasePool: _objc_autoreleasePoolPop(); _objc_autoreleasePoolPush();
	        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopBeforeWaiting);
	 
	        /// 7. sleep to wait msg.
	        mach_msg() -> mach_msg_trap();
	        
	 
	        /// 8. 通知Observers，线程被唤醒
	        __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopAfterWaiting);
	 
	        /// 9. 如果是被Timer唤醒的，回调Timer
	        __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__(timer);
	 
	        /// 9. 如果是被dispatch唤醒的，执行所有调用 dispatch_async 等方法放入main queue 的 block
	        __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(dispatched_block);
	 
	        /// 9. 如果如果Runloop是被 Source1 (基于port的) 的事件唤醒了，处理这个事件
	        __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__(source1);
	 
	 
	    } while (...);
	 
	    /// 10. 通知Observers，即将退出RunLoop
	    /// 此处有Observer释放AutoreleasePool: _objc_autoreleasePoolPop();
	    __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__(kCFRunLoopExit);
	}
	
###RunLoop 与 AutoReleasePool

App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。

第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush() 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。

###RunLoop 与定时器

NSTimer 其实就是 CFRunLoopTimerRef，他们之间是 toll-free bridged 的。一个 NSTimer 注册到 RunLoop 后，RunLoop 会为其重复的时间点注册好事件。例如 10:00, 10:10, 10:20 这几个时间点。RunLoop为了节省资源，并不会在非常准确的时间点回调这个Timer。Timer 有个属性叫做 Tolerance (宽容度)，标示了当时间点到后，容许有多少最大误差。

如果某个时间点被错过了，例如执行了一个很长的任务，则那个时间点的回调也会跳过去，不会延后执行。就比如等公交，如果 10:10 时我忙着玩手机错过了那个点的公交，那我只能等 10:20 这一趟了。

CADisplayLink 是一个和屏幕刷新率一致的定时器（但实际实现原理更复杂，和 NSTimer 并不一样，其内部实际是操作了一个 Source）。如果在两次屏幕刷新之间执行了一个长任务，那其中就会有一帧被跳过去（和 NSTimer 相似），造成界面卡顿的感觉。在快速滑动TableView时，即使一帧的卡顿也会让用户有所察觉。

###RunLoop 与 GCD

实际上 RunLoop 底层也会用到 GCD 的东西，比如 RunLoop 是用 dispatch_source_t 实现的 Timer。但同时 GCD 提供的某些接口也用到了 RunLoop， 例如 dispatch_async()。

当调用 dispatch_async(dispatch_get_main_queue(), block) 时，libDispatch 会向主线程的 RunLoop 发送消息，RunLoop会被唤醒，并从消息中取得这个 block，并在回调 \__CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE\__() 里执行这个 block。但这个逻辑仅限于 dispatch 到主线程，dispatch 到其他线程仍然是由 libDispatch 处理的。

dispatch_after 同理。

###RunLoop 与网络请求

通常使用 NSURLConnection 时，你会传入一个 Delegate，当调用了 [connection start] 后，这个 Delegate 就会不停收到事件回调。实际上，start 这个函数的内部会会获取 CurrentRunLoop，然后在其中的 DefaultMode 添加了4个 Source0 (即需要手动触发的Source)。CFMultiplexerSource 是负责各种 Delegate 回调的，CFHTTPCookieStorage 是处理各种 Cookie 的。

当开始网络传输时，我们可以看到 NSURLConnection 创建了两个新线程：com.apple.NSURLConnectionLoader 和 com.apple.CFSocket.private。其中 CFSocket 线程是处理底层 socket 连接的。NSURLConnectionLoader 这个线程内部会使用 RunLoop 来接收底层 socket 的事件，并通过之前添加的 Source0 通知到上层的 Delegate。

<img src=http://blog.ibireme.com/wp-content/uploads/2015/05/RunLoop_network.png width = 90%>

NSURLConnectionLoader 中的 RunLoop 通过一些基于 mach port 的 Source 接收来自底层 CFSocket 的通知。当收到通知后，其会在合适的时机向 CFMultiplexerSource 等 Source0 发送通知，同时唤醒 Delegate 线程的 RunLoop 来让其处理这些通知。CFMultiplexerSource 会在 Delegate 线程的 RunLoop 对 Delegate 执行实际的回调。

#####AFNetworking

AFURLConnectionOperation 这个类是基于 NSURLConnection 构建的，其希望能在后台线程接收 Delegate 回调。为此 AFNetworking 单独创建了一个线程，并在这个线程中启动了一个 RunLoop：


	+ (void)networkRequestThreadEntryPoint:(id)__unused object {
	    @autoreleasepool {
	        [[NSThread currentThread] setName:@"AFNetworking"];
	        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
	        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
	        [runLoop run];
	    }
	}
	 
	+ (NSThread *)networkRequestThread {
	    static NSThread *_networkRequestThread = nil;
	    static dispatch_once_t oncePredicate;
	    dispatch_once(&oncePredicate, ^{
	        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
	        [_networkRequestThread start];
	    });
	    return _networkRequestThread;
	}
	
RunLoop 启动前内部必须要有至少一个 Timer/Observer/Source，所以 AFNetworking 在 [runLoop run] 之前先创建了一个新的 NSMachPort 添加进去了。通常情况下，调用者需要持有这个 NSMachPort (mach_port) 并在外部线程通过这个 port 发送消息到 loop 内；但此处添加 port 只是为了让 RunLoop 不至于退出，并没有用于实际的发送消息。

	- (void)start {
	    [self.lock lock];
	    if ([self isCancelled]) {
	        [self performSelector:@selector(cancelConnection) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
	    } else if ([self isReady]) {
	        self.state = AFOperationExecutingState;
	        [self performSelector:@selector(operationDidStart) onThread:[[self class] networkRequestThread] withObject:nil waitUntilDone:NO modes:[self.runLoopModes allObjects]];
	    }
	    [self.lock unlock];
	}

当需要这个后台线程执行任务时，AFNetworking 通过调用 [NSObject performSelector:onThread:..] 将这个任务扔到了后台线程的 RunLoop 中。


以上就是这篇文章的全部内容，上面大部分的内容摘抄以及参考自:

* [深入理解 RunLoop](http://blog.ibireme.com/2015/05/18/runloop/)
* [优化UITableViewCell高度计算的那些事](http://blog.sunnyxx.com/2015/05/17/cell-height-calculation/)
* [SunnyXX](http://weibo.com/u/1364395395?from=feed&loc=nickname) 的[技术分享视频](http://yun.baidu.com/s/1eQvk3rO)

欢迎提出建议,个人联系方式详见 [关于](http://rannie.github.io/about)。






















