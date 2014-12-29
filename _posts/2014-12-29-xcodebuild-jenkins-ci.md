---
layout: post
title:  "使用 Xcodebuild + Jenkins + Apache 做 iOS 持续集成"
date:   2014-12-29 16:44:00
categories: iOS

---

这篇博客主要讲解如何用 xcodebuild 命令行打包，利用 Jenkins 做持续集成，以及使用 Mac 自带的 Apache 服务将 ipa 存放的目录分享出去。

###Xcodebuild 打包

只要安装了 Xcode , 就可以在命令行里使用 xcodebuild 命令进行打包。
假设我有个需要打包的工程，在个人目录下。