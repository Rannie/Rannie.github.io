
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












[1]:https://developer.apple.com/library/ios/documentation/UserExperience/Conceptual/LocationAwarenessPG/CoreLocation/CoreLocation.html


