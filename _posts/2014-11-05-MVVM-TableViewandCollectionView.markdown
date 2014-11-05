---
layout: post
title:  "MVVM实践--TableView，CollectionView的ViewModel（支持RAC）"
date:   2014-11-05 21:50:00
categories: iOS
---

<br />
最近对项目进行重构，内部讨论了下开始用MVVM+RAC的方式来架设项目，于是把所有代码推倒重来了一次。

###什么是MVVM

<br />
**MVVM**（Model-View-ViewModel）已经不算是一个很新的名词，最早是微软提出的一种架构，在iOS开发中，我们大部分时间都在与MVC打交道，而MVVM在iOS中将使得设计上将模型与视图解耦，将其交互逻辑放到Viewmodel上而不是传统的控制器中，这样也便于我们写出更简洁的Controller代码。

<br />
在objc.io的专题中，有一个专题专门讲述了如何写出更简洁的Controller，也有文章向我们介绍了MVVM，虽然描述和代码比较简单，不过也对理解有很大的帮助。
链接:[更简洁的controller][1], [MVVM介绍][2]


###什么是RAC

<br />
**RAC** (ReactiveCocoa) 是**GitHub**官方推出的一个采用函数式编程思想的iOS开发框架，在一些交互性比较强的应用中，函数式编程将大大的提高我们的编程效率和代码的可读性。在苹果推出的Swift语言中，也很大程度上采用了函数式编程思想对语言规则进行设计。

<br />
RAC的核心在于信号的传递以及处理，结合Objective-C中的block语法，所有的事件在程序中将用信号方式进行包装，传递。另外RAC也对UI层进行了方法的扩充（Category），尤其是对Target-Action模式进行了包装，如UIButton，UIGestureReconigzer等等都增加了RAC信号方法。

<br />
本人对于RAC的使用也局限于这几个月，更多的参考了一些更早使用RAC项目的源代码，以及一些使用者写的心得。比较有名的是花瓣网开发者的关于RAC的文章:

> * [ReactiveCocoa与Functional Reactive Programming][3]
> * [说说ReactiveCocoa 2][4]
> * [ReactiveCocoa2实战][5]


###传统TableView以及CollectionView的问题

<br />
为了方便陈述，我先把这两种视图统称为**列表视图**。列表视图在所有的移动端应用都是最常见的展示方式。

<br />
上面更简洁的Controllers文章也对传统的使用Tableview的方法进行了改造，众多的数据源和代理方法使得ViewController中有许多重复代码。于是我们想根据MVVM的设计思想决定将样板代码设计到一个类中去，让他将我们指定的datasource和tableview绑定，并自身成为数据源和代理，将传进来的样板Cell进行展示。


###RAC版的接口设计及思路

* HRTableViewModel.h

``` 
#import <UIKit/UIKit.h>
#import <ReactiveCocoa.h>

@interface HRTableViewModel : NSObject <UITableViewDataSource, UITableViewDelegate>
{
    UITableView     *_tableView;
    NSArray         *_data;
    UITableViewCell *_templateCell;
    RACCommand      *_disSelection;
}

+ (instancetype) bindingForTableView:(UITableView *)tableView
                        sourceSignal:(RACSignal *)source
                 didSelectionCommand:(RACCommand *)didSelection
                        templateCell:(UINib *)templateCellNib;

+ (instancetype) bindingForTableView:(UITableView *)tableView
                        sourceSignal:(RACSignal *)source
                 didSelectionCommand:(RACCommand *)didSelection
               templateCellClassName:(NSString *)classCell;

- (UITableViewCell *)dequeueCellAndBindInTable:(UITableView *)tableView indexPath:(NSIndexPath *)indexPath;

@end
	
```

两个类实例方法唯一不同的地方是前者传入的Cell为Nib模板，而后者为类，由于tableView都可以对它们进行注册后重用，所以对传入参数进行了区分。

<br />
注意到这里是TableView(RAC版本)的ViewModel，通过对数据源的订阅来完成数据的实时刷新:
		
```
		[source subscribeNext:^(id x) {
            self->_data = x;
            [self->_tableView reloadData];
        }];
```

重用Cell的方法实现为：

```
- (UITableViewCell *)dequeueCellAndBindInTable:(UITableView *)tableView indexPath:(NSIndexPath *)indexPath {
    id<BindViewDelegate> cell = [tableView dequeueReusableCellWithIdentifier:_templateCell.reuseIdentifier];
    [cell bindModel:_data[indexPath.row]];
    return (UITableViewCell *)cell;
}
```

这个*BindViewDelegate*是一个单独的协议文件，来规定Cell绑定_data中每个model的行为，具体的实现则有具体的cell展示和model数据来共同完成。

而这个方法单独列出来则是为了方便日后子类化定制时候，能够对这段代码进行复用，毕竟这个ViewModel只是简单的负责了展示，没有更多的逻辑来适应不同的情况和需求。

<br />
接着来看下CollectionView的ViewModel

* HRCollectionViewModel.h

```
@interface HRCollectionViewModel : NSObject <UICollectionViewDataSource, UICollectionViewDelegate>
{
    NSArray                 *_data;
    UICollectionView        *_collectionView;
    UICollectionViewCell    *_templateCell;
    RACCommand              *_selectCommand;
}

+ (instancetype)bindWithCollectionView:(UICollectionView *)collectionView
                            dataSource:(RACSignal *)source
                      selectionCommand:(RACCommand *)command
                          templateCell:(UINib *)nibCell;

+ (instancetype)bindWithCollectionView:(UICollectionView *)collectionView
                            dataSource:(RACSignal *)source
                      selectionCommand:(RACCommand *)command
                 templateCellClassName:(NSString *)classCell;

- (UICollectionViewCell *)dequeueCellAndBindInCollectionView:(UICollectionView *)collectionView indexPath:(NSIndexPath *)indexPath;

```

一般来说，CollectionView的使用方式和TableView类似，唯一的不同在于需要指定布局(Layout),设置一些参数来告诉系统如何展示每个Item。所以接口设计以及类的实现上，跟TableView大同小异。

###非RAC


正在编写中...稍后补充...


[1]: http://objccn.io/issue-1-1/
[2]: http://objccn.io/issue-13-1/
[3]: http://limboy.me/ios/2013/06/19/frp-reactivecocoa.html
[4]: http://limboy.me/ios/2013/12/27/reactivecocoa-2.html
[5]: http://limboy.me/tech/2014/06/06/deep-into-reactivecocoa2.html