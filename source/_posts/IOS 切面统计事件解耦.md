---
title: IOS 切面统计事件解耦
date: 2017-08-02 02:09:05
tags:
---

统计这个事情可以说是个巨无语的系统，当然不把他独立出来也就不是什么问题了，只是一堆牛皮癣似得代码穿插在项目各个地方，毕竟真正应用到一个app里的统计都跟业务有着很强的绑定关系，脱离业务的统计数据基本没什么大用，先吐槽一波再开始正文。。。。
![DingTalk20170802140347.png](http://upload-images.jianshu.io/upload_images/3994053-18807a49c709e49a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 基础封装
先从用第三方的来说，基本上就只是需要包个壳就ok了，建个manager，初始化sdk一封装，加几个常用统计方法基本ok。常见的方法就是传个event名再加个properties传扩展字典，用户登录状态绑定注销，通用字段增删改，基本上这就满足了大部分需求。。。

如果是纯自己手写，上面说的壳放着，剩下的仿照sdk来，基本功能要实现异步队列记录往本地写数据，定期上传，处理好读写关系是关键，这里不多说不是这篇重点。

ps:有个壳才敢放开手折腾优化

#### 切面封装
切面统计其实可以看我之前的[IOS 百行代码切面日志](http://www.jianshu.com/p/a9219e618ca2)，整个完成的就是切面的封装，看过的基本应该了解这套逻辑切面的时候不关心切面方法的参数的话会非常好用。
如果业务关联强的情况，虽然也能处理但要针对那些业务作出对应的逻辑，导致切面封装里夹杂很多特殊逻辑,下面的方法内部要对originAOP拆分取所有参，甚至要复制部分业务层逻辑过来最后完成一个统计。
```
-(void)al_logger:(id)log originAOP:(id)originAOP
```
所以我个人建议这里就封装些简单的少参甚至无参的统计，然后基本上简单的通用统计和业务统计建个类或者plist，列一下要切面的类和方法就完成了。

凡事都有特例，如果本身方法内多个参数完成一次统计就不多，那我之前的百行系列就已经足够你解耦了。

ps:郑重声明AOP 至少我现在还没发现IOS里怎么实现切面静态方法（类方法）
# 通用统计
通用统计就是可以脱离业务完全抽离的部分，这部分其实可以设计部分业务承接，比如通用点击事件可以把点击的UI对象扩展出一个字典的属性值，如此对于通用统计来说只是看看有没有某个属性有的话就扔到统计里，和业务层没有关联，但后面带来的好处很大！这个最后说。
#### App生命周期统计
应用生命周期的统计随便建个类，监听下面的通知就ok了，当然这个类就不能销毁了，比较懒得做法直接AOPLogger 类扩展一下init里监听，因为我本身的类里连init方法都没重写。
<!-- more -->
UIApplicationDidFinishLaunchingNotification （通知名称） --->   application:didFinishLaunchingWithOptions:(委托方法）：在应用程序启动后直接进行应用程序级编码的主要方式。

UIApplicationWillResignActiveNotification(通知名称）--->applicationWillResignActive:（委托方法）：用户按下主屏幕按钮调用 ，不要在此方法中假设将进入后台状态，只是一种临时变化，最终将恢复到活动状态

UIApplicationDidBecomActiveNotification（通知名称） ---->applicationDidBecomeActive:(委托方法）：应用程序按下主屏幕按钮后想要将应用程序切换到前台时调用，应用程序启动时也会调用，可以在其中添加一些应用程序初始化代码

UIApplicationDidEnterBackgroundNotification(通知名称）----->applicationDidEnterBackground:（委托方法）：应用程序在此方法中释放所有可在以后重新创建的资源，保存所有用户数据，关闭网络连接等。如果需要，也可以在这里请求在后台运行更长时间。如果在这里花费了太长时间（超过5秒），系统将断定应用程序的行为异常并终止他。

UIApplicationWillEnterForegroundNotification(通知名称） ---->applicationWillEnterForeground:(委托方法):当应用程序在applicationDidEnterBackground:花费了太长时间，终止后，应该实现此方法来重新创建在applicationDidEnterBackground中销毁的内容，比如重新加载用户数据、重新建立网络连接等。

UIApplicationWllTerminateNotification（通知名称） ----> applicationWillTerminate:(委托方法):现在很少使用，只有在应用程序已进入后台，并且系统出于某种原因决定跳过暂停状态并终止应用程序时，才会真正调用它。

#### 点击事件（不包括手势）
这里点击事件切面分为3类
1.UIControl的addTarget触发，UIButton都会走UIApplication里的这方法，但需要忽略一部分，不然会和第二个部分重叠，而且手势addTarget不走，不过手势不走正好可以区分手势
```
-(void)sendAction:(SEL)action to:(id)target from:(id)sender forEvent:(UIEvent *)event;
```
2.设置delegate委托式系统点击触发，这里直接切了UICollectionView，UITableView，UITabBarController，UITabBar，UIAlertView，UIActionSheet这几个setDelegate方法，然后加标识判断切委托方法，用Aspects库的方法swizzing因为他不允许重复swizzing算双保险。

3.UIAlertAction切block属性的set方法，每次setBlock的时候替换成我自己的block，在我的block内执行设置的block。

下面就是协议，用pod的话AOPLogger/AOPClick就有了，用的时候类扩展一下AOPLogger遵守协议实现对应方法
```
@protocol AOPLoggerClickProtocol <NSObject>

@optional
- (void)alcp_sendAction:(SEL)action to:(id)target from:(id)sender forEvent:(UIEvent *)event;
- (void)alcp_customIgnore_sendAction:(SEL)action to:(id)target from:(id)sender forEvent:(UIEvent *)event;
- (void)alcp_collectionView:(UICollectionView *)collectionView didSelectItemAtIndexPath:(NSIndexPath *)indexPath from:(id)sender;
- (void)alcp_tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath from:(id)sender;
- (void)alcp_tabBarController:(UITabBarController *)tabBarController didSelectViewController:(UIViewController *)viewController from:(id)sender;
- (void)alcp_tabBar:(UITabBar *)tabBar didSelectItem:(UITabBarItem *)item from:(id)sender;
- (void)alcp_alertView:(UIAlertView *)alertView clickedButtonAtIndex:(NSInteger)buttonIndex from:(id)sender;
- (void)alcp_actionSheet:(UIActionSheet *)actionSheet clickedButtonAtIndex:(NSInteger)buttonIndex from:(id)sender;
- (void)alcp_alertControllerAction:(UIAlertAction *)action from:(id)sender;

@end
```
ps:覆盖可能不全，漏了什么欢迎git提pull request

#### 手势统计
这个是切了UIView的addGestureRecognizer方法，每次添加手势的时候我多加个自己AOPLogger的taget-action到gestureRecognizer对象上，这样手势统计就搞定了。
如果要用这个统计点击和长按，判断一下状态是结束的时候统计gestureRecognizer类型是UITapGestureRecognizer,UILongPressGestureRecognizer就行，其实手势的统计或许都在结束的时候统计区分下类型就够了吧！
```
@protocol AOPLoggerGestureRecognizerProtocol <NSObject>

@optional
-(void)algrp_handleGesture:(UIGestureRecognizer*)gestureRecognizer;

@end
```
#### 页面统计
这个太寻常了，感觉没什么好说，使用方法如上，之所以还提供个pod的AOPLogger/AOPPageView,主要是如果要统计浏览时长逻辑，自己方便实现（如显示的时候runtime随便塞个date进去，消失的时候算一下上报），还有我默认预先忽略了部分页面,省了自己写。
```
@protocol AOPLoggerPageViewProtocol <NSObject>

@optional
-(BOOL)alpvp_viewIgnore:(id)sender;
-(void)alpvp_viewDidAppear:(BOOL)animated sender:(id)sender;
-(void)alpvp_viewDidDisappear:(BOOL)animated sender:(id)sender;

@end
```
# 业务统计
业务统计首先放的位置要保证在对应业务线的目录里，毕竟跟业务关联密切，大的业务逻辑调整的时候看一下放业务统计目录下对应的地方需不需要修改。
#### 无参数方法统计
这一类就是需要写在业务层里但统计的时候只记录一个Event名，比如有个方法-(void)abc:(id)a b:(id)b c:(id)c;我只是记录他被调用，这种直接AOPLogger调用下[AOPLogger AOPLoggerWithClassString:classString methodString:@"abc:b:c" log:@"abc"]或扔plist里默认直接解决。
#### 方法内逻辑处理统计
这个就有点厉害了，比如下订单的时候我买了一堆东西里面带着各种商品属性，要你把某种属性的商品做统计，只是举例...这时候直接写这个类的类扩展load方法里切面然后写逻辑处理统计，这里不建议用Aspects库，底层保证切一个类的一个方法一次是好用的，可业务层各种奇葩逻辑都会有，想解耦经常用奇招所以最好允许多次切面，dispatch_once保证自己的逻辑需要的切一次就可以了，可以用我简单封装的方法。
```
@interface NSObject (AOPLogger)

+(void)al_hookOrAddWithOriginSeletor:(SEL)originalSelector swizzledSelector:(SEL)swizzledSelector error:(NSError**)error;

@end
```
#### 前后逻辑关联统计
这个可以结合点击来讲，比上面还要让人头疼，类似统计我查看一个东西，要记录我在哪点击的查看以及当前页面的数据属性，还有从哪来到当前页面的，要查看东西的id。这个时候就需要结合通用统计来搞了，我们在最底层可以做的是把本身点击的UI对象的默认字典属性和最上层页面的默认字典属性以及点击对象target上层UI对象的默认字典属性做统计，最上层的激活页面这个是可以在任何地方搞出来的，回头我可以把类扩展放上来，在任何页面都可以拿到当前页面，前一个页面以及页面属于第几层。底层在响应默认点击事件之前处理我们的统计逻辑，接下来就是怎么把属性赋值，这个时候就需要我们去切面当前页面里做model赋值的方法然后同时把想要统计的属性赋值到当前页面的默认字典属性里，如果这是cell上的button那么赋值cell的时候就可以切面。

总的来说就是寻找UI对象赋值点，然后切面为UI对象添加统计属性，我们runtime加到所有UI对象上隐藏的默认字典属性字段专门承接关联信息，底层只做读取整合，业务层切面赋值。

这个时候就会发现数据对象与UI对象绑定协议的制定会直接决定实现的复杂度。
#### 后序
业务层的统计我个人认为原则上尽量后端做这个事，毕竟要保证统计数据的正确性，其实减少前端统计很重要，毕竟现在web（pc网页），wap（手机网页），小程序，安卓，ios一牵扯业务逻辑统计非常需要统一校准，不然数据绝对超乎你的想象。
# 归纳统计解耦思路
统计放到其他端其实解耦思路也是大致相同。。。下面总结：
1.找底层触发的事件点
2.业务层找赋值点
3.使用AOP的思路对上面的地方进行插入我们的统计逻辑
