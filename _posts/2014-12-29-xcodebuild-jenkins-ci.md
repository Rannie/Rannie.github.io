---
layout: post
title:  "使用 Xcodebuild + Jenkins + Apache 做 iOS 持续集成"
date:   2014-12-29 16:44:00
categories: iOS

---

这篇博客主要讲解如何用 xcodebuild 命令行打包，利用 Jenkins 做持续集成，以及使用 Mac 自带的 Apache 服务将 ipa 存放的目录分享出去。

###Xcodebuild 打包


####命令的基础使用

只要安装了 Xcode , 就可以在命令行里使用 xcodebuild 命令进行打包。
假设我有个需要打包的工程 *BuildDemo* ，在个人( *mark* )目录下。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014122901.png)

我现在想把它打包成 ipa 格式的文件则需要打开终端。
首先进入项目:
	
	cd BuildDemo/

打包成 .xcarchive 到 buildserver 目录下

	xcodebuild -archivePath "/Users/mark/buildserver/BuildDemo.xcarchive" -workspace BuildDemo.xcworkspace -sdk iphoneos -scheme "BuildDemo" -configuration "Debug" archive
	
由于我这个项目只是个简单的 project 并没有 workspace 所以命令需要去掉 *-workspace BuildDemo.xcworkspace* 这个参数。执行后会产生 log ， 下图为一部分

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014122903.png)

打包后可以看到指定文件夹下有我们刚才生成的 xcarchive 文件

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014122904.png)

然后我们开始打包成 ipa 文件
使用命令

	xcodebuild -exportArchive -exportFormat IPA -archivePath "/Users/mark/buildserver/BuildDemo.xcarchive" -exportPath "/Users/mark/buildserver/BuildDemo.ipa"
	
执行后依旧生成 log ， 最后是 \** EXPORT SUCCEEDED **

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014122905.png)

使用 xcodebuild 生成 ipa 文件就完成了。

####指定 Configuration 和配置文件来生成 IPA 文件

有的时候我们不仅仅是简单的设置 *configuration* 为 *Debug* 或 *Release* ，可能还有 AdHoc 版啊， Inhouse 版去给用户或者测试使用。这时候需要我们在项目中去配置更多的 *configuration* 。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014122906.png)

点击箭头所在的地方我们可以添加比如 Debug.Inhouse Release.Adhoc Release.Inhouse 

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014122907.png)

我们需要在 *target* 的 *buildsettings* 中去指定不同的配置，比如代码签名和配置文件

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014122908.png)

或者是针对不同版本指定不同的宏， *BundleID* ， *BundleDisplayName* 等等。

然后我们使用 xcodebuild 时可以生成不同版本的 xcarchive 以及 ipa 文件了，举个例子，比如我想根据 Adhoc 的配置去打包则命令如下

	xcodebuild -archivePath "/Users/mark/buildserver/BuildDemo.Adhoc.xcarchive" -sdk iphoneos -scheme "BuildDemo" -configuration "Release.Adhoc" archive

	xcodebuild -exportArchive -exportFormat IPA -exportProvisioningProfile "指定的配置文件名称" -archivePath "/Users/mark/buildserver/BuildDemo.Adhoc.xcarchive" -exportPath "/Users/mark/buildserver/BuildDemo.Adhoc.ipa"
	
打包完成

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014122910.png)

>> **注意**： 如果你的 workspace 使用了 *CocoaPods* 那么需要也为你的 Pods 工程添加 *configuration* ，下图为我为其他 workspace 添加的 *configuration* 。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014123001.png)

### Jenkins CI

####Java 环境

Jenkins 需要 Java 环境来运行，如果没有需要自行下载。

*安装 JDK 8 可能会遇到的问题* ：

安装 JDK 8 可能在一开始提示: "This Installer is supported only on OS X 10.7.3 or Later",这是由于安装包中的一个函数发生了错误，我们需要将函数修改后重新打包。

流程:

1. 打开下载好的 jdk 安装包的 DMG .这时候你会在 finder 左侧能看到已经被挂上了。
2. bash 中运行：pkgutil --expand /Volumes/JDK\ 8\ Update\ 05/JDK\ 8\ Update\ 05.pkg /tmp/jdk8.unpkg (解释：通过 pkgutil 命令把刚刚下载好的 dmg 解压开来，存放到 /tmp/jdk8.unpkg 这个目录中去。)
3. 走入到 /tmp/jdk8.unpkg 目录中去。你可以通过 finder 也可以通过终端命令进入。
4. 找到目录下的 Distribution 文件，用vim或者是编辑器 (open -e) 打开。
5. cmd+f找到里面的 pm_install_check 这个函数。

		function pm_install_check() {
		  if(!(checkForMacOSX('10.7.3') == true)) {
		    my.result.title = 'OS X Lion required';
		    my.result.message = 'This Installer is supported only on OS X 10.7.3 or Later.';
		    my.result.type = 'Fatal';
		    return false;
		  }
		  return true;
		}
	
	修改成：
	
		function pm_install_check() {
		  return true;
		}
		
	保存。

6. 然后我们重新打包。命令如下：
	pkgutil --flatten /tmp/jdk8.unpkg/ /tmp/jdk8.pkg
7. 打开 /tmp/jdk8.pkg 文件。
	open /tmp/jdk8.pkg 或者是从 finder 中找到并点击打开，你就会发现可以正常安装了。
	
####下载运行配置 Jenkins

去 Jenkins [官网][1]下载 Web Archive (.war)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014123002.png)

下载后进入该 .war 目录执行 

	java -jar jenkins.war --httpPort=9000
	
这个 http 端口 9000 可以根据情况自由指定。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014123003.png)



### Apache 分享文件目录



















[1]:http://jenkins-ci.org/
