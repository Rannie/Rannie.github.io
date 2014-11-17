---
layout: post
title:  "为应用做iOS8,iPhone6以及6plus的适配"
date:   2014-11-17 16:40:00
categories: iOS

---

###新的屏幕
iPhone6，6plus的出现让我们在开发中不再局限于*320x480*了，我们需要在布局界面时考虑更多的适配问题。贴上一张[PaintCode][1]的图解

![screenimage](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014111701.png)

图中是以点为单位，像素x2即可。
在Xcode6中，如果对以前的项目进行编译,默认是不会对项目进行适配6,6plus的屏幕的，xcode默认对原项目进行拉伸显示在新设备上（可能会比较丑），我们可以通过手动设置launchScreenFile设置。而且图片设置我们可以设置3x图片来适应更高的分辨率。

![sizeimage](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014111702.png)

关于手动适配的建议:

>> 只要手动指定了启动图或者那个xib，屏幕分辨率就已经变成应有的大小了，老代码中所有关于写死frame值的代码会让你适配很头疼，如果去手动适配就要全部适配，建议在找到个可行方案前先不要做修改，自动适配方案还算不影响使用。
面对4个分辨率的iPhone，建议使用Auto Layout布局 + Image Assets管理各个分辨率的图片 + Interface Builder（xib+storyboard）构建UI，Size Classes在低版本iOS系统的表现未知。想要这套手动适配方案，起码你的工程需要部署在iOS6+，还不用AutoLayout布局的会死的蛮惨。

###尺寸以及设备信息

做一些横竖屏适配的应用（比如视频播放）可能会发现一个问题，就是当你横屏读取**[UIScreen mainScreen].bounds**时，跟竖屏时的数值不同，比如竖屏时width=375,height=667,竖屏时就会变为width=667,height=375。而在之前，这个数值是始终不会变的，所以在横竖屏切换时需要注意这个问题。我做的处理是，在程序启动时即收集设备各种信息，存放在内存中。在将来收集反馈，横竖屏尺寸变化以及读设备宽高宏时从内存读取。

	- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    	self.deviceInfo = [DeviceInfo info];
    	[self.deviceInfo gatherDeviceInfomation];
    	return YES;
	}

收集信息通过*gatherDeviceInfomation*方法收集

	- (void)gatherDeviceInfomation {
	    self.screenBounds = [UIScreen mainScreen].bounds;
	    self.screenSize = self.screenBounds.size;
	    self.screenWidth = self.screenSize.width;
	    self.screenHeight = self.screenSize.height;
	    self.systemVersion = [[[UIDevice currentDevice] systemVersion] floatValue];
	    
	    struct utsname systemInfo;
	    uname(&systemInfo);
	    NSString *code = [NSString stringWithCString:systemInfo.machine encoding:NSUTF8StringEncoding];
	    self.deviceVersion = (DeviceVersion)[[DeviceInfo deviceNamesByCode][code] integerValue];
	    
	    if (StringEqual(code, @"x86_64") || StringEqual(code, @"i386")) {
	        code = @"Simulator";
	    }
	    self.deviceName = code;
	}
	
*DeviceVersion*是一个描述设备的枚举，细节可以看[Demo][2]

当我们在启动程序时收集了设备信息后，就可以通过定义类似这样的宏

	#define SCREEN_WIDTH ([DeviceInfo info].screenWidth)
	
来全局访问设备信息了，而且之后也不会因为横竖屏而改变。

###Autolayout?Masonry?



###sizeclass, xib+autolayout



###远程推送



[1]:http://www.paintcodeapp.com/news/iphone-6-screens-demystified
[2]:https://github.com/Rannie/make-app-adaptive-to-ios8-ip6-6plus