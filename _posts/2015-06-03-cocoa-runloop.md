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

####RunLoop 与线程的关系

线程和 RunLoop 之间是一一对应的，其关系是保存在一个全局的 Dictionary 里。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 RunLoop（主线程除外）。

####RunLoop 成员类们

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

























