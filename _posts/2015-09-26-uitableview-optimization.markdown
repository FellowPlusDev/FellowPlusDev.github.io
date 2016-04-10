---
layout: post
title: UITableView 性能优化
author: Binboy
tags: iOS
date: 2015-09-26 10:12:24.000000000 +08:00
---

面试的时候遇到这个问题，竟一时没有全答上来，于是Google了一下，常见的一些譬如`Cell重用`、`设计统一Cell`、`缓存Cell高度`，`Cell数据资源缓存`，这些其实平时都在用，但因为平时还是缺乏总结，回答这么个问题的时候却只想到说“==重用==”、“==缓存==”，道理你都懂，但这样极度概括的答案在面试过程中并不是什么好答案，深有体会~

另外，也有自己平时很少用而想不起来的，就是性能要求更高一些的话Cell中用到的视图控件可以尽可能自行`drawRect`。

面试结果也未可知，便先吃一堑长一智，趁热将其总结总结。

##Cell重用机制

这是TableView的基本使用，仅作简单归纳。

    [tableView dequeueReusableCellWithIdentifier:(NSString *)identifier forIndexPath:(NSIndexPath *)indexPath]; 

使用这个方法需要提前注册Cell到对应的tableView：

* 1、Stroyboard: 定义Cell的Prototype，并设置其Reusable Identifier

![](http://binboy.top/content/images/2015/10/Cell_Prototype.png)

>这里值得一提的是：Cell的Prototype最好高度抽象统一，越少越好，因为每个不同的Prototype都有其对应的Cell重用池，创建的Cell就越多，重用效率也就越低。（详见以下设计统一规格的Cell）

* 2、Xib自定义: 

        [registerNib:(nullable UINib *)nib forCellReuseIdentifier:(NSString *)identifier];

* 3、代码自定义：

        [registerClass:(nullable Class)cellClass forCellReuseIdentifier:(NSString *)identifier];

##设计统一规格的Cell

统一Cell的规格，不仅能减少设计不同Cell所需要代码量和nib文件，更重要的是能提高Cell的重用率，提升TableView整体性能。

* 等高的Cell好设计，显示不同数据就可以了，无须多费篇幅。
* 动态计算高度的Cell也应该统一设计，比如下面这个赤兔的例子

![](http://binboy.top/content/images/2015/10/Cell_Demo.png)

>虽然看起来有些不一样，但实际上整体构成是统一的，都是由头像、姓名、公司、职位、时间、内容、其他（图片、网页链接、视频等附件）以及转发、赞、和评论三个底部按钮，这些控件的显示可根据模型中的不同数据在代码当中动态控制就好了。

##创建ViewModel，计算并储存Cell的UI尺寸信息

    @interface BYTweetViewModel : NSObject
    
    @property(strong, nonatomic) BYTweetModel *dataModel; //原始数据模型

    @property(assign, nonatomic) CGFloat cellHeight; //Cell 高度

    - (void)calculateCellHeight; //计算高度

    @end

这里有个坑需要`注意`：

>在iOS中，系统是先调用“tableView:heightForRowAtIndexPath:”获取每个Cell即将显示的高度，确定整个UITableView的布局。然后才调用“tableView:cellForRowAtIndexPath”获取Cell。因此，使用了ViewModel来保存UI信息，Cell高度的计算和使用的时机需要特别留意。

##提前处理Cell需要显示的数据资源

在Cell显示之前，将从服务器加载获取到的原始数据在ViewModel中进行提前处理，一般包括图片的加载和压缩、富文本的多样化显示（NSString->NSAttributeString）。

这时ViewModel可能会是这样

    @interface BYTweetViewModel : NSObject
    
    @property(strong, nonatomic) BYTweetModel *dataModel; //原始数据模型

    @property(assign, nonatomic) CGFloat cellHeight; //Cell 高度

    //需要显示的数据内容
    @property(strong, nonatomic) NSAttributeString *titleToShow;
    @property(strong, nonatomic) NSAttributeString *contentToShow;

    - (void)calculateCellHeight; //计算高度

    - (void)handleSourceDataModel;

    @end

##其它
我了解的，也是常用的方案基本是以上几种了，总之呢，还是可以回到我面试时候的高度概括，尽可能“重用”、“缓存”，用空间换取时间。

另外还有些更为极致的一些方式和操作细节也就不深入展开了，大致整合一下。

* Cell中的view尽可能不要使用透明
* 减少子视图的层级关系
* 图片载入在后台进程进行，滚出可视范围的载入进程cancel掉
* 图片资源尽可能使用PNG
* ……

##参考

知乎上有个讨论，阐述了许多各种各样不同的思路，虽然很少情况需要那么极致，但也可以在必要的时候尝试看看,不过相应的代码量增加了，可一定要注意避免各种莫名bug出现哦~

[知乎：如何加强 iOS 里的列表滚动时的顺畅感？](http://www.zhihu.com/question/20382396)
