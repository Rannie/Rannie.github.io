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
	
由于我这个项目只是个简单的 project 并没有 workspace 所以命令需要去掉 *-workspace BuildDemo.xcworkspace* 这个参数。执行后会产生好多 log ， 下图为一部分

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014122903.png)

打包后可以看到指定文件夹下有我们刚才生成的 xcarchive 文件

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014122904.png)

然后我们开始打包成 ipa 文件
使用命令

	xcodebuild -exportArchive -exportFormat IPA -archivePath "/Users/mark/buildserver/BuildDemo.xcarchive" -exportPath "/Users/mark/buildserver/BuildDemo.ipa"