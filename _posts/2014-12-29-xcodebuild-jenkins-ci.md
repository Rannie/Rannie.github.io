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
	
####下载运行 Jenkins

去 Jenkins [官网][1]下载 Web Archive (.war)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014123002.png)

下载后进入该 .war 目录执行 

	java -jar jenkins.war --httpPort=9000
	
这个 http 端口 9000 可以根据情况自由指定。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014123003.png)

看到红色标识的 log 标明 Jenkins 服务已经运行。

打开浏览器，输入 localhost:9000 可以看到

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014123004.png)


####配置服务

一开始我们可以做一些基础配置

点击左侧的*系统管理*

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014123005.png)

最上面有个执行者数量表示可以同时执行的 Build 数，同时 Build 同一个项目可能会发生一些问题，我会将其更改为1，其他触发的 Build 则会加入到队列中等待之前的 Build 完成才开始构建。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014123006.png)

这里设置 Jenkins 的 URL 以及系统管理员的邮箱。在公司里可能我们需要一台机器专门进行持续集成，那么需要这台机器手动指定 IP ，邮箱则是可以在构建失败的时候，将失败的 Log 发送到指定的开发人员邮箱中，让开发人员得知 svn 或者 git 等代码 Build 会有问题。不过我们需要指定 SMTP 地址，系统管理员的邮箱也需要开通 SMTP 服务。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2014123007.png)

设置完成后可以勾选 *通过发送测试邮件测试配置* 来先尝试发送一个测试邮件。

####添加一个构建

在首页我们点击*添加一个新的构建*

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2015010201.png)

填写构建的名称及类型，这里勾选的是自由风格的软件项目。创建完成后进入该构建的详细配置界面。我们首先要指定要构建的项目的根目录，以及是否使用 SVN 或者 GIT 进行源码管理。如果指定 svn 的话，每次构建将会先使用你指定的步骤去 svn 进行 svn up (或者其他你指定的步骤) 对代码进行更新后再进行构建。 

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2015010214.png)

然后我们可以选择执行的触发条件，比如你想每周一到周五，早上九点到下午六点，每二十分钟进行一次触发构建，那么可以按下面进行填写。具体的语法格式可以点击右侧的问号查询。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2015010215.png)

注意的是，如果采用了源码管理，如果这个期间没有版本更新，则不会继续进行构建。

下面添加构建步骤， Mac 下我们则选择 shell 来执行步骤命令。然后将我们之前的 xcodebuild 指令直接拷贝到这个步骤里就可以，两步即可完成 ipa 的生成。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2015010202.png)

如果你想执行一些定制化的构建，可以使用一些自己编写或开源的 shell 脚本，比如 Facebook 的 xctool : [https://github.com/facebook/xctool][2]

在构建中我们可以加入一些参数。点击 command 下的查看环境变量的参数列表。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2015010205.png)

可以看到有 build_id 对应构建时间， svn 版本号等等。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2015010203.png)

然后是添加邮件通知。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2015010204.png)

每次构建失败时会发送构建失败的邮件到指定邮箱中。之后保存即可。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2015010206.png)

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2015010207.png)

保存后我们对这个项目进行立即构建，很快发现构建完成时状态是红色，这次构建失败了。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2015010208.png)

进入这次构建，选择查看控制台输出可以查看整个构建过程的日志。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2015010209.png)

同时，这次失败的构建也会给我们发送邮件。查看邮件中的日志我们发现项目中并不存在 "BuildDemo.xcworkspace"

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2015010210.png)

原来在命令中多了一个 workspace 的参数，而这个项目中并不存在，于是把这个参数删除并再次进行构建。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2015010211.png)

完成后发现在指定目录下多了 BuildDemo_2.ipa 的文件。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2015010212.png)

同时 jenkins 会发送一个构建回复正常的邮件到之前指定的邮箱中。

![screenshot](https://raw.github.com/Rannie/Rannie.github.io/master/images/2015010213.png)

Jenkins 基础的配置及构建就进行完成，由于支持中文，使用起来并不困难，其余的功能可以自己进行了解。

### Apache 分享文件目录

####启动 Apache

Mac 下自带 Apache 服务，所以并不需要很多的步骤，只要打开终端 $ sudo apachectl start 即可以启动。

暂停 : sudo apachectl stop
重启 : sudo apachectl restart

####分享目录

如果你的系统是 Yosemite ，则需要对 apache 的 httpconfig 进行一些修改才可以分享个人目录。

/etc/apache2/httpd.conf 找到下面的行去掉前面的＃

\# LoadModule php5_module libexec/apache2/libphp5.so <br />
\# LoadModule userdir_module libexec/apache2/mod_userdir.so <br />
\# Include /private/etc/apache2/extra/httpd-userdir.conf <br />
\# LoadModule authz_core_module libexec/apache2/mod_authz_core.so <br />
\# LoadModule authz_host_module libexec/apache2/mod_authz_host.so <br />

/etc/apache2/extra/httpd-userdir.conf  找到下面的行去掉前面的＃

\# Include /private/etc/apache2/users/*.conf

如果已经去掉则不用处理，这样我们就可以通过 localhost/~$user_name 来访问个人目录了。

如果没有权限更改上述文件，简单的可以先拖到桌面上来进行修改然后放回之前位置覆盖即可。












[1]:http://jenkins-ci.org/
[2]:https://github.com/facebook/xctool