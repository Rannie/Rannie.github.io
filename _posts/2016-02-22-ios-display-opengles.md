---
layout: post
title:  "OpenGL ES 基础以及 iOS 设备渲染探究优化"
date:   2016-02-22 19:46:00
categories: iOS

---


客户端的开发，无非离不开数据和展示，而展示这个方面，首当其冲的就是视图、动画的渲染，切换等等。而且在用户的使用中，UI 是这个 APP 的门面，无论功能有多强大，体验不好也是无法留住用户的。


### 硬件图像显示的基本原理

![crt](http://blog.ibireme.com/wp-content/uploads/2015/11/ios_screen_scan.png)

首先从过去的 CRT 显示器原理说起。CRT 的电子枪按照上面方式，从上到下一行行扫描，扫描完成后显示器就呈现一帧画面，随后电子枪回到初始位置继续下一次扫描。为了把显示器的显示过程和系统的视频控制器进行同步，显示器（或者其他硬件）会用硬件时钟产生一系列的定时信号。当电子枪换到新的一行，准备进行扫描时，显示器会发出一个水平同步信号（horizonal synchronization），简称 HSync；而当一帧画面绘制完成后，电子枪回复到原位，准备画下一帧前，显示器会发出一个垂直同步信号（vertical synchronization），简称 VSync。显示器通常以固定频率进行刷新，这个刷新率就是 VSync 信号产生的频率。尽管现在的设备大都是液晶显示屏了，但原理仍然没有变。

![display](http://blog.ibireme.com/wp-content/uploads/2015/11/ios_screen_display.png)

通常来说，计算机系统中 CPU、GPU、显示器是以上面这种方式协同工作的。CPU 计算好显示内容提交到 GPU，GPU 渲染完成后将渲染结果放入帧缓冲区，随后视频控制器会按照 VSync 信号逐行读取帧缓冲区的数据，经过可能的数模转换传递给显示器显示。

在最简单的情况下，帧缓冲区只有一个，这时帧缓冲区（*后面会讲到缓冲区以及帧缓冲区的概念*）的读取和刷新都都会有比较大的效率问题。为了解决效率问题，显示系统通常会引入两个缓冲区，即双缓冲机制。在这种情况下，GPU 会预先渲染好一帧放入一个缓冲区内，让视频控制器读取，当下一帧渲染好后，GPU 会直接把视频控制器的指针指向第二个缓冲器。如此一来效率会有很大的提升。

双缓冲虽然能解决效率问题，但会引入一个新的问题。当视频控制器还未读取完成时，即屏幕内容刚显示一半时，GPU 将新的一帧内容提交到帧缓冲区并把两个缓冲区进行交换后，视频控制器就会把新的一帧数据的下半段显示到屏幕上，造成画面撕裂现象，如下图：

![frame](http://blog.ibireme.com/wp-content/uploads/2015/11/ios_vsync_off.jpg)

为了解决这个问题，GPU 通常有一个机制叫做**垂直同步**（简写也是 V-Sync），当开启垂直同步后，GPU 会等待显示器的 VSync 信号发出后，才进行新的一帧渲染和缓冲区更新。这样能解决画面撕裂现象，也增加了画面流畅度，但需要消费更多的计算资源，也会带来部分延迟。

那么目前主流的移动设备是什么情况呢？从网上查到的资料可以知道，iOS 设备会始终使用双缓存，并开启垂直同步。而安卓设备直到 4.1 版本，Google 才开始引入这种机制，目前安卓系统是三缓存+垂直同步。


### OpenGL ES

OpenGL ES 是一种软件技术，上文讲到计算机中 CPU GPU 协同工作完成渲染，在手机端，正是 OpenGL ES 横跨在两个处理器之间，协调两个区域之间的数据交换。

OpenGL ES 为了提升渲染的性能，为两个内存区域间的数据交换定义了 **缓冲区** 的概念 (buffers) 。缓冲区是指 GPU 能够控制和管理的连续 RAM 。程序从 CPU 的内存复制数据到 OpenGL ES 的缓冲区。通过独占缓冲区，GPU 能够尽可能以有效的方式读写内存。 GPU 把它处理数据的能力异步地应用在缓冲区上，意味着 GPU 使用缓冲区中的数据工作的同事，运行在 CPU 中的程序可以继续执行。

GPU 需要知道内存中的那个位置来存储渲染出来的 2D 图像像素数据，接收渲染结果的缓冲区称为 **帧缓冲区 (frame buffer)** 。渲染指令会在适当的时候替换帧缓冲区中的内容，OpenGL ES 会根据特定平台硬件配置和功能设置数据类型和偏移。通常来说，渲染结果可以存储到任意数量的 frame buffer 中。上面提到的双缓冲的两个缓冲称之为 **前帧缓冲区 (front frame buffer)** 和 **后帧缓冲区 (back frame buffer)** 。

在 OpenGL ES 中，所有的图像都可以由点，线段和三角形构成，所以 OpenGL ES 只渲染这三种图形。在接收到一些顶点数据后，经过**顶点着色器 (vertex shader)** 处理，装配输出给**片元着色器 (fragment shader)** ,再经过一些操作最终输出给帧缓冲区。什么是片元呢？通常在顶点着色器输出几何图形数据后，会进行**光栅化 (rasterizing)** 将这些形状数据转换为帧缓存中的颜色像素，而每一个颜色像素就叫做**片元 (fragment)** 。

下图为整个 OpenGL ES 的绘制管道:

![pipeline](https://raw.githubusercontent.com/Rannie/Rannie.github.io/master/images/2016-02/opengles_pipeline.png)


### iOS 设备与 OpenGL ES

每一个 iOS 原生用户界面对象都有对应的 Core Animation Layer, layer 会保存所有绘制操作的结果。苹果的 Core Animation 合成器使用 OpenGL ES 来尽可能高效地控制 GPU 、混合 layer 和切换帧缓冲区。图形程序员经常使用**混合 (composite)** 来描述混合图像来形成一个合成结果的过程。所有显示的图画都是通过 Core Animation 合成器来完成的，因此最终都涉及 OpenGL ES 。

苹果官方文档描述了 iOS 设备图形显示的架构:

![ios-structure](http://i.imgur.com/QcQlN7B.png)

##### 渲染的各个阶段

在应用内部有四个阶段:

* 布局：在这个阶段，程序设置 View/Layer 的层级信息，设置 layer 的属性，如 frame，background color 等等。
* 创建 backing image：在这个阶段程序会创建 layer 的 backing image，无论是通过 setContents 将一个 image 传給 layer，还是通过 drawRect：或 drawLayer:inContext：来画出来的。所以 drawRect：等函数是在这个阶段被调用的。
* 准备：在这个阶段，Core Animation 框架准备要渲染的 layer 的各种属性数据，以及要做的动画的参数，准备传递給 render server。同时在这个阶段也会解压要渲染的 image。（除了用 imageNamed：方法从 bundle 加载的 image 会立刻解压之外，其他的比如直接从硬盘读入，或者从网络上下载的 image 不会立刻解压，只有在真正要渲染的时候才会解压）。
* 提交：在这个阶段，Core Animation 打包 layer 的信息以及需要做的动画的参数，通过 IPC（inter-Process Communication）传递給 render server。

在应用外部有两个阶段:

当这些数据到达 render server 后，会被反序列化成 render tree。然后 render server 会做下面的两件事：

* 根据 layer 的各种属性（如果是动画的，会计算动画 layer 的属性的中间值），用 OpenGL 准备渲染。
* 渲染这些可视的 layer 到屏幕。

如果做动画的话，最后的两个步骤会一直重复知道动画结束。

我们都知道 iOS 设备的屏幕刷新频率是 60HZ。如果上面的这些步骤在一个刷新周期之内无法做完（1/60s），就会造成掉帧。

##### 资源消耗的原因和解决方案

相对于 CPU 来说，GPU 能干的事情比较单一：接收提交的纹理（Texture）和顶点描述（三角形），应用变换（transform）、混合并渲染，然后输出到屏幕上。通常你所能看到的内容，主要也就是纹理（图片）和形状（三角模拟的矢量图形）两类。

**纹理的渲染**

所有的 Bitmap，包括图片、文本、栅格化的内容，最终都要由内存提交到显存，绑定为 GPU Texture。不论是提交到显存的过程，还是 GPU 调整和渲染 Texture 的过程，都要消耗不少 GPU 资源。当在较短时间显示大量图片时（比如 TableView 存在非常多的图片并且快速滑动时），CPU 占用率很低，GPU 占用非常高，界面仍然会掉帧。避免这种情况的方法只能是尽量减少在短时间内大量图片的显示，尽可能将多张图片合成为一张进行显示。

当图片过大，超过 GPU 的最大纹理尺寸时，图片需要先由 CPU 进行预处理，这对 CPU 和 GPU 都会带来额外的资源消耗。目前来说，iPhone 4S 以上机型，纹理尺寸上限都是 4096x4096，更详细的资料可以看这里：iosres.com。所以，尽量不要让图片和视图的大小超过这个值。

**视图的混合 (Composing)**

当多个视图（或者说 CALayer）重叠在一起显示时，GPU 会首先把他们混合到一起。如果视图结构过于复杂，混合的过程也会消耗很多 GPU 资源。为了减轻这种情况的 GPU 消耗，应用应当尽量减少视图数量和层次，并在不透明的视图里标明 opaque 属性以避免无用的 Alpha 通道合成。当然，这也可以用上面的方法，把多个视图预先渲染为一张图片来显示。

**图形的生成。**

CALayer 的 border、圆角、阴影、遮罩（mask），CASharpLayer 的矢量图形显示，通常会触发离屏渲染（offscreen rendering），而离屏渲染通常发生在 GPU 中。当一个列表视图中出现大量圆角的 CALayer，并且快速滑动时，可以观察到 GPU 资源已经占满，而 CPU 资源消耗很少。这时界面仍然能正常滑动，但平均帧数会降到很低。为了避免这种情况，可以尝试开启 CALayer.shouldRasterize 属性，但这会把原本离屏渲染的操作转嫁到 CPU 上去。对于只需要圆角的某些场合，也可以用一张已经绘制好的圆角图片覆盖到原本视图上面来模拟相同的视觉效果。最彻底的解决办法，就是把需要显示的图形在后台线程绘制为图片，避免使用圆角、阴影、遮罩等属性。

##### 离屏渲染

OpenGL 中，GPU 屏幕渲染有以下两种方式：

* On-Screen Rendering
意为当前屏幕渲染，指的是 GPU 的渲染操作是在当前用于显示的屏幕缓冲区中进行。

* Off-Screen Rendering
意为离屏渲染，指的是 GPU 在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作。

相比于当前屏幕渲染，离屏渲染的代价是很高的，主要体现在两个方面：

* 创建新缓冲区
要想进行离屏渲染，首先要创建一个新的缓冲区。

* 上下文切换
离屏渲染的整个过程，需要多次切换上下文环境：先是从当前屏幕（On-Screen）切换到离屏（Off-Screen）；等到离屏渲染结束以后，将离屏缓冲区的渲染结果显示到屏幕上有需要将上下文环境从离屏切换到当前屏幕。而上下文环境的切换是要付出很大代价的。

所以在图形生成的步骤我们要尽可能的避免离屏渲染，或者开启 *shouldRasterize* 属性。

### 渲染优化的注意点

根据上面的描述，简单的汇总一些优化渲染的注意点：

隐藏的绘制：catextlayer 和 uilabel 都是将 text 画入 backing image 的。如果改了一个包含 text 的 view 的 frame 的话，text 会被重新绘制。

Rasterize：当使用 layer 的 shouldRasterize 的时候（记得设置适当的 layer 的 rasterizationScale），layer 会被强制绘制到一个 offscreen image 上，并且会被缓存起来。这种方法可以用来缓存绘制耗时（比如有比较绚的效果）但是不经常改的 layer，如果 layer 经常变，就不适合用。

离屏绘制： 使用 Rounded corner， layer masks， drop shadows 的效果可以使用 stretchable images。比如实现 rounded corner，可以将一个圆形的图片赋值于 layer 的 content 的属性。并且设置好 contentsCenter 和 contentScale 属性。

Blending and Overdraw ：如果一个 layer 被另一个 layer 完全遮盖，GPU 会做优化不渲染被遮盖的 layer，但是计算一个 layer 是否被另一个 layer 完全遮盖是很耗 cpu 的。将几个半透明的 layer 的 color 融合在一起也是很消耗的。

我们要做的：

设置 view 的 backgroundColor 为一个固定的，不透明的 color。
如果一个 view 是不透明的，设置 opaque 属性为 YES。（直接告诉程序这个是不透明的，而不是让程序去计算）
这样会减少 blending 和 overdraw。

如果使用 image 的话，尽量避免设置 image 的 alpha 为透明的，如果一些效果需要几个图片融合而成，就让设计用一张图画好，不要让程序在运行的时候去动态的融合。

这篇文章林林总总的描述了一些概念和注意点，在优化时可以结合 Instrument 中的 Core Animation 和 GPU Driver 来进行测试。对于局部需要特别要求性能的地方可以尝试 Facebook 开源的 [AsyncDisplayKit](https://github.com/facebook/AsyncDisplayKit) 或者国内大牛的 [YYKit](https://github.com/ibireme/YYKit) 。

以上就是这篇文章的全部内容，上面大部分的内容总结以及参考自一些书籍和文章:

* 《Learning OpenGL ES for iOS》 - Erik M. Buck
* 《OpenGL ES 2.0 Programming Guide》 - Aaftab Munshi / Dan Ginsburg / Dave Shreiner 
* 《iOS 视图、动画渲染机制探究》 - 腾讯 Bugly 
* 《iOS 保持界面流畅的技巧》 - ibireme 

欢迎提出建议,个人联系方式详见 [关于](http://rannie.github.io/about)。
