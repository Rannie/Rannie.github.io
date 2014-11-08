---
layout: post
title:  "MVVM实践--TableView，CollectionView的BindingHelper（支持RAC）"
date:   2014-11-05 21:50:00
categories: iOS
---

最近对项目进行重构，内部讨论了下开始用MVVM+RAC的方式来架设项目，于是把所有代码推倒重来了一次。

###什么是MVVM

**MVVM**（Model-View-ViewModel）已经不算是一个很新的名词，最早是微软提出的一种架构，在iOS开发中，我们大部分时间都在与MVC打交道，而MVVM在iOS中将使得设计上将模型与视图解耦，将其数据逻辑放到*viewModel*上而不是传统的控制器中，这样也便于我们写出更简洁的Controller代码。

在objc.io的专题中，有一个专题专门讲述了如何写出更简洁的Controller，也有文章向我们介绍了MVVM，虽然描述和代码比较简单，不过也对理解有很大的帮助。
链接:[更简洁的controller][1], [MVVM介绍][2]

###什么是RAC

**RAC** (ReactiveCocoa) 是**GitHub**官方推出的一个采用函数式编程思想的iOS开发框架，在一些交互性比较强的应用中，函数式编程将大大的提高我们的编程效率和代码的可读性。在苹果推出的Swift语言中，也很大程度上采用了函数式编程思想对语言规则进行设计。

RAC的核心在于信号的传递以及处理，结合Objective-C中的block语法，所有的事件在程序中将用信号方式进行包装，传递。另外RAC也对UI层进行了方法的扩充（Category），尤其是对Target-Action模式进行了包装，如UIButton，UIGestureReconigzer等等都增加了RAC信号方法。

本人对于RAC的使用也局限于这几个月，更多的参考了一些更早使用RAC项目的源代码，以及一些使用者写的心得。比较有名的是花瓣网开发者的关于RAC的文章:

> * [ReactiveCocoa与Functional Reactive Programming][3]
> * [说说ReactiveCocoa 2][4]
> * [ReactiveCocoa2实战][5]


###传统TableView以及CollectionView的问题

为了方便陈述，我先把这两种视图统称为**列表视图**。列表视图在所有的移动端应用都是最常见的展示方式。

上面更简洁的Controllers文章也对传统的使用Tableview的设计进行了改造，众多的数据源和代理方法使得ViewController中有许多重复代码。于是我想根据逻辑抽象的原则决定将样板代码设计到一个类中去，让他将我们指定的datasource和tableview绑定，并自身成为数据源和代理，将传进来的样板Cell进行展示。

###RAC版的接口设计及思路

项目链接：[Rannie/HRTableCollectionBindingHelper][6]

* HRTableViewBindingHelper.h




		#import <UIKit/UIKit.h>
		#import <ReactiveCocoa.h>
		
		@interface HRTableViewBindingHelper : NSObject <UITableViewDataSource, UITableViewDelegate>
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
	


两个类实例方法唯一不同的地方是前者传入的Cell为Nib模板，而后者为类，由于tableView都可以对它们进行注册后重用，所以对传入参数进行了区分。

注意到这里是TableView(RAC版本)的BindingHelper，通过对数据源的订阅来完成数据的实时刷新:


		[source subscribeNext:^(id x) {
	           self->_data = x;
	           [self->_tableView reloadData];
	       }];
        


重用Cell的方法实现为：



	- (UITableViewCell *)dequeueCellAndBindInTable:(UITableView *)tableView indexPath:(NSIndexPath *)indexPath {
	    id<BindViewDelegate> cell = [tableView dequeueReusableCellWithIdentifier:_templateCell.reuseIdentifier];
	    [cell bindModel:_data[indexPath.row]];
	    return (UITableViewCell *)cell;
	}



这个*BindViewDelegate*是一个单独的协议文件，来规定Cell绑定_data中每个model的行为，具体的实现则有具体的cell展示和model数据来共同完成。

而这个方法单独列出来则是为了方便日后子类化定制时候，能够对这段代码进行复用，毕竟这个BindingHelper只是简单的负责了展示，没有更多的逻辑来适应不同的情况和需求。

接着来看下CollectionView的BindingHelper

* HRCollectionViewBindingHelper.h



		@interface HRCollectionViewBindingHelper : NSObject <UICollectionViewDataSource, UICollectionViewDelegate>
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



一般来说，CollectionView的使用方式和TableView类似，唯一的不同在于需要指定布局(Layout),设置一些参数来告诉系统如何展示每个Item。所以接口设计以及类的实现上，跟TableView大同小异。

###非ReactiveCocoa版


TableViewBindingHelper扩充的接口:

	+ (instancetype)bindingForTableView:(UITableView *)tableView
                        sourceList:(NSArray *)source
                 didSelectionBlock:(TableSelectionBlock)block
                      templateCell:(UINib *)templateCellNib;

	+ (instancetype)bindingForTableView:(UITableView *)tableView
	                         sourceList:(NSArray *)source
	                  didSelectionBlock:(TableSelectionBlock)block
	              templateCellClassName:(NSString *)templateCellClass;
	
	- (void)reloadDataWithSourceList:(NSArray *)source;


这里使用Array取代了RAC版的Signal，另外使用block来对点击Cell的事件进行回调。区别在于使用Array以后每次需要刷新时需要手动调用 *- reloadData...* 方法。而CollectionView增加的接口与此类似，则不赘述了。

###与MVVM结合对Controller进行改造

下面通过一个简单的例子来将BindingHelper和ViewModel融入到Controller中。

我们有一个 **User** 的模型，并使用 **UITableView** 对其数据列表进行展示。

首先，创建一个**ViewModel**来获取数据。

	- (void)fetchUsers {
	    NSInteger count = arc4random()%20 + 1;
	    
	    NSMutableArray *list = [NSMutableArray array];
	    for (int i = 0; i < count; i++) {
	        User *user = [User new];
	        user.name = [NSString stringWithFormat:@"user--%d", i];
	        user.age = arc4random()%100;
	        [list addObject:user];
	    }
	    
	    self.userList = list;
	}

将创建的列表引用给 **ViewModel** 的 *userList* 属性。
在视图控制器中的 *viewDidLoad* 创建 viewModel 以及 bindingHelper，并用 *RACObserver* 来观察 *userList* 的变化。

	- (void)viewDidLoad {
		  [super viewDidLoad];
		  self.viewModel = [[UserViewModel alloc] init];
		  
		  UINib *cellNib = [UINib nibWithNibName:@"UserCell" bundle:nil];
		  RACCommand *command = [[RACCommand alloc] initWithSignalBlock:^RACSignal *(id input) {
		      NSLog(@"select : %@", input);
		      return [RACSignal empty];
		  }];
		  self.bindingHelper = [HRTableViewBindingHelper bindingForTableView:self.tableView sourceSignal:RACObserve(self.viewModel, userList) didSelectionCommand:command templateCell:cellNib];
	}

获取数据只要调用 *fetchUsers* 方法列表即可刷新了。

一个简单的结合Demo就完成了。在很多场景，我们可能有更多的逻辑针对TableView的展示等，我们可以通过子类BindingHelper的方式来实现。

再贴完整项目地址:[HRTableCollectionBindingHelperDemo][6]

欢迎提出建议，个人联系方式详见[关于][7]。


[1]: http://objccn.io/issue-1-1/
[2]: http://objccn.io/issue-13-1/
[3]: http://limboy.me/ios/2013/06/19/frp-reactivecocoa.html
[4]: http://limboy.me/ios/2013/12/27/reactivecocoa-2.html
[5]: http://limboy.me/tech/2014/06/06/deep-into-reactivecocoa2.html
[6]: https://github.com/Rannie/HRTableCollectionBindingHelper
[7]: http://rannie.github.io/about/