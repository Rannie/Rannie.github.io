---
layout: post
title:  "Location and Maps Programming Guide 要点"
date:   2014-12-14 21:27:00
categories: iOS

---


出于对 iBeacon 的学习诉求，这两天阅读了下苹果官方文档关于定位的部分。这篇博客主要将其中的重要部分取出来用中文叙述。
如果想更详细的了解，还是要看原版文档：[Location and Maps Programming Guide][1]

###获取用户的位置

苹果提供了两种获取用户位置的服务：

* 基本的位置服务 —— 持续跟踪用户位置，并刷新数据。
* 显著位置变化服务 —— 节约能耗，只在用户位置有显著变化时才对地理数据进行更新。

在需要后台跟踪用户位置时，需要打开 Capabilities 中的 **BackgroundModes** . 并勾选 **Location Updates** 。 

由于用户可能主动关闭位置服务。建议在启动前调用 *[CLLocationManager locationServicesEnabled]* 方法来了解是否开启定位服务，如果返回 NO 而且你试图开始定位的话，系统会提示用户是否启用定位服务。

####启动基本定位服务 (Standard Location Service)

启动服务流程: <br />
1. 启动一个 *CLLocationManager* 实例。 <br />
2. 配置 *desiredAccuracy* 和 *distanceFilter* 属性。 <br />
3. 指定委托对象 *id<CLLocationManagerDelegate>* 。 <br />
4. 调用 *startUpdatingLocation* 开始进行获取数据。 <br />

停止则调用 *stopUpdatingLocation* 方法。

**iOS8 适配** <br />
iOS8 之后，*CLLocationManager* 新增了两个方法 *requestAlwaysAuthorization* , *requestWhenInUseAuthorization* 。正如方法名表达的意思一样，是允许应用总是访问用户位置，或者是只有应用使用时才访问用户位置，如果不添加其中一个方法的话， 不会回调接收数据的方法。在添加方法的同时需要给 Info.plist 中添加两个提示用户的提示语键-值: *NSLocationAlwaysUsageDescription* , *NSLocationWhenInUseUsageDescription* 。

####启动显著位置变化服务 (Significant-Change Location Service)

该项服务使用 Wi-Fi 来确定用户的位置，并报告给应用。显著位置变化的服务甚至可以唤醒后台运行或者没有运行的 iOS 程序来提供数据。启动服务的步骤相似于基本定位服务，不同的是调用 *startMonitoringSignificantLocationChanges* 方法即可，不需要配置精度等属性。

停止则调用 *stopMonitoringSignificantLocationChanges* 。

如果你在挂起或者退出程序时运行该项服务，这项服务会自动唤醒你的程序当新的位置数据到达。在唤醒时，应用会置入后台，并且获得大约10s的时间来重启位置服务和处理位置数据(您必须在后台手动重启位置服务，关于何时启动，参见下一节)。由于你的程序在后台，你必须做尽可能少的工作和避免任何任务（例如访问网络）在分配的时间到时之前。如果没有做完你的应用也会被终止掉。如果一个应用需要更多的时间来处理位置数据，它可以使用 *UIApplication* 的 *beginBackgroundTaskWithName:expirationHandler:* 方法。

注意：如果用户禁止了后台应用刷新设置，显著位置变化服务不会重新启动你的应用程序。此外，如果禁止了这个选项，应用在前台也不会收到显著位置变化或者区域监测事件。

####从服务接收位置数据

成功回调: *locationManager:didUpdateLocations:* <br />
失败回调: *locationManager:didFailWithError:*

回调时你可以得到位置信息例如时间戳(timeStamp)或者测量精确度等。

####何时开启位置服务

直到应用需要位置信息时我们才需要开启位置服务。如果在不需要定位时寻求用户启动位置服务可能会失去用户对你应用的信任，另外一种建立信任的方式是在 Infn.plist 中添加 *NSLocationUsageDescription* 键并设值用于向用户描述你如何使用位置信息。

如果你在监测区域或者使用显著位置变化服务，在有些情况下，你必须在启动应用同时启动位置服务。应用使用这些服务可以在挂起或者未运行情况下重启并接收位置数据。**不过**，尽管应用是重新启动，定位服务并不会自动启动。当一个应用是由于位置更新而重启时，*application:willFinishLaunchingWithOptions:* 或 *application:didFinishLaunchingWithOptions:* 的启动 option 字典会包含一个 *UIApplicationLaunchOptionsLocationKey* 键，该键的存在用于提示你新的位置数据正等待你去处理。为了获得这些数据，你必须建立一个新的 *CLLocationManager* 来重启之前已经终止的位置服务，当你重新启动了这些服务， locationManager 会通过代理将位置数据传递给你。

###后台获取位置事件

iOS 支持交付地点事件给后台挂起或者终止的应用。你有多个选项来后台获取位置数据，各有利弊。尽可能的使用显著位置服务来进行定位，如果需要的话，也可以配置后台模式的应用来使用基本定位服务。**提示**：由于用户可能会禁用后台运行模式，所以你可以由 *UIApplication* 的 *backgroundRefreshStatus* 属性来确定是否可以在后台处理位置更新。

####在后台使用基本定位服务

需要配置项目的 Capabilities 打开 BackgroundMode 并勾选 LocationUpdates 。这个在文章开头有提到过。不过在后台时，尽可能的不要做太多工作来处理新的数据。

基于后台运行可能会耗费电力，你需要对你的定位进行额外的管理：

* 确保 *location manager* 的 *pausesLocationUpdatesAutomatically* 属性是 YES , 这样 Core Location 会暂停位置更新(关闭定位硬件)当需要这么做的时候，比如用户不会移动时。（当它不会确定一个位置时，也会暂停）
* 分配一个适当的值给 *location manager* 的 *activityType* 属性。此属性有助于确定何时可以安全的暂停位置更新。对于一个导航程序，把该属性设为 *CLActivityTypeAutomotiveNavigation* 会导致只有用户在一段时间内没有发生显著的位置变化时才会暂停位置更新事件。
* 调用 *allowDeferredLocationUpdatesUntilTraveled:timeout:* 方法来尽可能的推迟交付位置数据的时间。具体参见章节<程序在后台时推迟位置更新>

当 *location manager* 暂停位置更新，会调用代理方法 *locationManagerDidPauseLocationUpdates:* ，继续更新则调用 *locationManagerDidResumeLocationUpdates:* .可以使用这些方法来调整应用程序的行为。比如，当位置更新暂停，你可以使用这个代理通知来持久化数据或者完全停止位置更新。而导航应用则应该询问用户，是否导航应该暂停使用。

####程序在后台时推迟位置更新

在 iOS6 及以后，你可以在应用在后台时推迟位置更新数据的交付。推荐使用这个功能的原因是可以没有任何问题的处理定位数据。推迟更新可以让你的应用睡眠更长的时间。由于推迟位置更新需要有 GPS 硬件的存在，所以要调用 *CLLocationManager* 的类方法 *deferredLocationUpdatesAvailable* 来确定是否支持推迟更新。

调用推迟的示例：

	- (void)locationManager:(CLLocationManager *)manager	      didUpdateLocations:(NSArray *)locations {		   // Add the new locations to the hike		   [self.hike addLocations:locations];		   // Defer updates until the user hikes a certain distance		   // or when a certain amount of time has passed.		   if (!self.deferringUpdates) {		      CLLocationDistance distance = self.hike.goal - self.hike.distance;		      NSTimeInterval time = [self.nextAudible timeIntervalSinceNow];		      [locationManager allowDeferredLocationUpdatesUntilTraveled:distance		   	  self.deferringUpdates = YES;		   } 		}

当你在调用 *allowDeferredLocationUpdatesUntilTraveled:timeout:* 方法指定的条件满足时， *location manager* 会回调 *locationManager:didFinishDeferredUpdatesWithError:* 来通知你已经停止了推迟数据交付。 *location mananger* 会确保回调该方法的次数与应用调用推迟方法的次数相同。推迟结束后，还是会通过 *locationManager:didUpdateLocations:* 来更新位置数据。

您也可以通过调用 *CLLocationManager* 类的 *disallowDeferredLocationUpdates* 方法停止位置更新推迟。当你调用这个方法，或者干脆使用 *stopUpdatingLocation* 方法停止位置更新, *location manager* 会调用代理方法 *locationManager:didFinishDeferredUpdatesWithError:* 来通知你推迟被停止了。

如果定位出错，会返回一些 *CLError* 参数来报告错误信息，若想了解可查 Core Location 参考中的 CLError 常量。











[1]:https://developer.apple.com/library/ios/documentation/UserExperience/Conceptual/LocationAwarenessPG/CoreLocation/CoreLocation.html


