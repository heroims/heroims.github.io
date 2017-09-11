---
title: IOS组件化与工程管理
date: 2017-02-05 02:09:05
tags:
    - IOS
    - 组件化
    - 工程管理
    - 架构设计
---

谈及组件化其实网上也有不少文章了，但我个人认为不结合工程管理去单讲组件化恐怕很难让人理解概念，而去实践的时候也只是照猫画虎。
# 工程管理
组件化的实现很重要的一个组成部分应该是工程拆分，这里我的方案是采取git管理项目pod管理依赖很常见很普通的方法。
理想的状态每一个模块都是独立的，可以单独拿出来测试，发布，也就是每一个子模块其实都是一个git仓库，这里紧接着就是子仓库和主项目的关系问题，上边说到了git和pod，还有一个submodule我喜欢用这个来做子仓库的管理上边为啥没提它呢？
因为submodule本身就是git自带的就是git的一部分，常用命令有
``` 
# 添加子仓库
git submodule add 仓库地址 路径
#初始化所有子仓库
git submodule init
# 更新子仓库
git submodule update

# 也可以初始化更新一起
git submodule update --init
```

pod只是帮我把依赖关系理清直接本地pod，因为坑爹的开发阶段难免有互相block的情况，那边东西弄完了，但还没有做发布还不稳定，但另一边已经急着要看一眼整体调用的效果了。。。
当然这时候也可以让对面先打个beta的tag，那样可想而知最后会有多少没用的tag，另一方面就是bug联调恢复节点排查的时候，另一方估计只有一个方案就是回滚上一个tag中间哪的问题一点点打tag联调。
上边说的打tag也只是个例子，当然你可以改Podfile对应不同branch，但那也是要每次调都要改一下的，但我这种模式由于是本地pod所以podfile不用动了每次都去指向对应的项目，剩下的就是对子仓库随意切换branch甚至commit节点都可以，调ok了直接commit一下submodule指向的更改即可。然后另一方更新一下再pod update把依赖关系重新建立一下（如果没有添加或删除，甚至这步都不需要，本地pod引用目录我们submodule的本地文件夹里面有什么变化这边自动会变，添加删除是因为依赖关系发生变化了所以跟着需要重新建立）,当然理想情况回头有空单独整合一套submodule和pod的命令，submodule更新时判断有增删操作执行pod update其他情况不处理。

这里紧接着就是公共库的处理以及怎么去建立主仓库与子仓库的依赖，我这里把基本思路给出，具体情况还是自己再改动，这里首先就是建立podspec来提供依赖建立
```
Pod::Spec.new do |s|
s.name                  = 'MainWorkSpaceDemo'
s.version               = '1.0.0'
s.summary               = 'A new container controller to slide  '
s.homepage              = 'github.com'
s.license               = { :type => 'MIT', :file => 'README.md' }
s.author                = { 'heroims' => 'heroims@163.com' }
s.source                = { :git => '', :tag => "#{s.version}" }
s.platform              = :ios, '5.0'
s.source_files          = 'ZYQRouter/*.{h,m}'
s.requires_arc          = true
#公共仓库
s.subspec 'BaseTool' do |ss|
ss.source_files = 'ZYQRouter/*.{h,m}'
end
#模块1
s.subspec 'Module1' do |sss|
sss.source_files = 'Module1/Module1Lib/*.{h,m}'
end
#模块2
s.subspec 'Module2' do |ssss|
ssss.source_files = 'Module2/Module2Lib/*.{h,m}'
end

end
```
看见上边相比就明白了把，开发的时候最好要作为模块给人的东西放在一个目录下，当然不放也可以，这里就是为了方便
然后就是引用了,下面是module1工程的，只引用了一个公共库，真正开发的时候则会引用很多，然后build测试模块的app给测试，提供给主仓库的东西放在事先约定的目录下，其他的随便看心情，反正对别人没影响就是自己爽不爽
```
target 'Module1' do
# Uncomment the next line if you're using Swift or would like to use dynamic frameworks
# use_frameworks!

# Pods for Module1
pod 'MainWorkSpaceDemo/BaseTool', :path => '../MainWorkSpaceDemo.podspec'

target 'Module1Tests' do
inherit! :search_paths
# Pods for testing
end

target 'Module1UITests' do
inherit! :search_paths
# Pods for testing
end

end
```
<!-- more -->
就此基本的项目依赖思路构建就算讲完了，剩下的就是调用了，其实上边的东西你掌握好了，组件化就已经可以单用这种工程管理模式解决了
下面的就是大家能经常搜到的一些组件化的东西，在我看来剩下的只是锦上添花，工程管理好组件化才有真正意义。从开发的角度上不论开发什么用什么语言，组件化或者说模块化通用思路就是分离多个仓库然后自动化建立依赖关系，项目工程里互相调用适当的用反射的方法实现调用，基本上每一个小仓库就都能独立的运行了。
顺便放一下我的ZYQRouter：https://github.com/heroims/ZYQRouter
里面的demo虽然没用submodule但基本可以阐述完整这套东西。
正式开始讲代码里的架构，ZYQRouter主要是方便各个模块之间的互相调用。之所以重复造轮子，其实只是自己项目需要，另外就是想完善这套组件化的实现。
这里ZYQRouter分为页面路由和方法路由，页面路由负责根据URL做各页面跳转甚至远程调度方法路由，方法路由则是提供target-action实例方法调用和invokeSelectorObjects反射调用静态方法，目的就是让各个模块开发过程中不引用对方的情况下也可独立按约定调用对方模块运行调试自己的相关内容，大家都开发完各个单元测试ok，集成到主项目里就可以基本跑通，当然现实是联调还是会通常出些小问题，但没什么大碍。
# 页面路由
关于页面路由如下，用过蘑菇街Router的看这个会很亲切，我只是在它的基础上添加了重定向这个功能，这重定向的由来一个是动态更新页面跳转逻辑方便，另一个就是我们自己的需求客服系统里。。。。你会发现一个订单链接地址由客服发来，网页上用这链接用户打开的就是网页自己订单，客服打开就是客服系统该用户的订单，app上用户打开就是用户订单页面，于是救星就是重定向，把xxx.xxx.xxx/crm-order/orderid和xxx.xxx.xxx/order/orderid都重定向到applink://order/orderid，还有就是订单有大改动的时候
则是xxx.xxx.xxx/crm-order/orderid和applink://order/orderid重定向到xxx.xxx.xxx/order/orderid直接开网页用户订单，还有很多奇葩需求全靠重定向这救命稻草，所以这个重定向真的很实用。

顺便再说下注册的事，因为我的Router里提供了target-action的调用所以上面说的远程调度target-action可以用一个url如applink://target-action/:target/:action?xxx=xxx完成，只用注册applink://target-action/:target/:action内部调用target-action方法。
而让所有部门全依照这一个逻辑规则产出链接简直天方夜谭，前端放在网页上的链接按这样估计一堆人吐槽，但仅仅ios部门之间按照这一规则跑还是可以的。
但当然有比较折中的方法，毕竟注册太多url也占地啊，这时候神奇的重定向就又可以上线救援了，如xxx.xxx.xxx/order?xxx=xxx这类直接重定向xxx.xxx.xxx/order到applink://target-action/ordertarget/orderaction这就好了，你注册的就可以少点但前提是你的target-action里处理的情况多。
另外写页面路由最好根据模块单独创建相应的类，比如Module1里可以单独的建个Module1PageFactory，有个方法-(void)openModule1VC1WithO1:(id)o1 o2:(id)o2 o3:(id)o3类似方法然后+(void)load里注册Router调用open的方法，这样开发阶段用方法路由，而在需要从外部进入时采用页面路由方式也就是URL方式


```Objective-C
/**
重定向 URLPattern 到对应的 newURLPattern 
@param URLPattern 原scheme
@param newURLPattern 新scheme
*/
+ (void)redirectURLPattern:(NSString *)URLPattern toURLPattern:(NSString*)newURLPattern;

/**
*  注册 URLPattern 对应的 Handler，在 handler 中可以初始化 VC，然后对 VC 做各种操作
*
*  @param URLPattern 带上 scheme，如 applink://beauty/:id
*  @param handler    该 block 会传一个字典，包含了注册的 URL 中对应的变量。
*                    假如注册的 URL 为 applink://beauty/:id 那么，就会传一个 @{@"id": 4} 这样的字典过来
*/
+ (void)registerURLPattern:(NSString *)URLPattern toHandler:(ZYQRouterHandler)handler;

/**
*  注册 URLPattern 对应的 ObjectHandler，需要返回一个 object 给调用方
*
*  @param URLPattern 带上 scheme，如 applink://beauty/:id
*  @param handler    该 block 会传一个字典，包含了注册的 URL 中对应的变量。
*                    假如注册的 URL 为 applink://beauty/:id 那么，就会传一个 @{@"id": 4} 这样的字典过来
*                    自带的 key 为 @"url" 和 @"completion" (如果有的话)
*/
+ (void)registerURLPattern:(NSString *)URLPattern toObjectHandler:(ZYQRouterObjectHandler)handler;

/**
*  取消注册某个 URL Pattern
*
*  @param URLPattern
*/
+ (void)deregisterURLPattern:(NSString *)URLPattern;

/**
*  打开此 URL
*  会在已注册的 URL -> Handler 中寻找，如果找到，则执行 Handler
*
*  @param URL 带 Scheme，如 applink://beauty/3
*/
+ (void)openURL:(NSString *)URL;

/**
*  打开此 URL，同时当操作完成时，执行额外的代码
*
*  @param URL        带 Scheme 的 URL，如 applink://beauty/4
*  @param completion URL 处理完成后的 callback，完成的判定跟具体的业务相关
*/
+ (void)openURL:(NSString *)URL completion:(void (^)(id result))completion;

/**
*  打开此 URL，带上附加信息，同时当操作完成时，执行额外的代码
*
*  @param URL        带 Scheme 的 URL，如 applink://beauty/4
*  @param parameters 附加参数
*  @param completion URL 处理完成后的 callback，完成的判定跟具体的业务相关
*/
+ (void)openURL:(NSString *)URL withUserInfo:(NSDictionary *)userInfo completion:(void (^)(id result))completion;

/**
* 查找谁对某个 URL 感兴趣，如果有的话，返回一个 object
*
*  @param URL
*/
+ (id)objectForURL:(NSString *)URL;

/**
* 查找谁对某个 URL 感兴趣，如果有的话，返回一个 object
*
*  @param URL
*  @param userInfo
*/
+ (id)objectForURL:(NSString *)URL withUserInfo:(NSDictionary *)userInfo;

/**
*  是否可以打开URL
*
*  @param URL
*
*  @return
*/
+ (BOOL)canOpenURL:(NSString *)URL;

/**
*  调用此方法来拼接 urlpattern 和 parameters
*
*  #define ROUTE_BEAUTY @"beauty/:id"
*  [ZYQRouter generateURLWithPattern:ROUTE_BEAUTY, @[@13]];
*
*
*  @param pattern    url pattern 比如 @"beauty/:id"
*  @param parameters 一个数组，数量要跟 pattern 里的变量一致
*
*  @return
*/
+ (NSString *)generateURLWithPattern:(NSString *)pattern parameters:(NSArray *)parameters;
```

# 方法路由
关于方法路由如下,target-action模式就是自动根据class来alloc init初始化完target对象，然后@selector把那action方法调用了返回，而静态方法则是runtime搞定，日常需求基本满足，但还有点缺陷注释里已说明，由于invokeSelectorObjects根据className和selectorName调用静态方法所以封装成了C方法，另外就是这个不常用算是尝试。
```Objective-C
/**
*
*  调度工程内的组件方法
*  [ZYQRouter performTarget:@"xxxClass" action:@"xxxxActionWithObj1:obj2:obj3" objects:obj1,obj2,obj3,nil]
*  内部自动 alloc init 初始化对象
*
*  @param targetName    执行方法的类
*  @param actionName    方法名
*  @param object1,... 不定参数 不支持C基本类型
*
*  @return 方法回参
*/
+ (id)performTarget:(NSString*)targetName action:(NSString*)actionName objects:(id)object1,...;

/**
*
*  调度工程内的组件方法
*  [ZYQRouter performTarget:@"xxxClass" action:@"xxxxActionWithObj1:obj2:obj3" shouldCacheTaget:YES objects:obj1,obj2,obj3,nil]
*  内部自动 alloc init 初始化对象
*
*  @param targetName    执行方法的类
*  @param actionName    方法名
*  @param shouldCacheTaget   设置target缓存
*  @param object1,... 不定参数 不支持C基本类型
*
*  @return 方法回参
*/
+ (id)performTarget:(NSString*)targetName action:(NSString*)actionName shouldCacheTaget:(BOOL)shouldCacheTaget objects:(id)object1,...;

/**
*
*  调度工程内的组件方法
*  [ZYQRouter performTarget:@"xxxClass" action:@"xxxxActionWithObj1:obj2:obj3" shouldCacheTaget:YES objects:obj1,obj2,obj3,nil]
*  内部自动 alloc init 初始化对象
*
*  @param targetName    执行方法的类
*  @param actionName    方法名
*  @param shouldCacheTaget   设置target缓存
*  @param objectsArr   参数数组 不支持C基本类型
*
*  @return 方法回参
*/
+ (id)performTarget:(NSString*)targetName action:(NSString*)actionName shouldCacheTaget:(BOOL)shouldCacheTaget objectsArr:(NSArray*)objectsArr;

/**
*
*  添加未找到Target 或 Action 逻辑
*
*  @param notFoundHandler    未找到方法回调
*  @param targetName    类名
*
*  @return
*/
+ (void)addNotFoundHandler:(ZYQNotFoundTargetActionHandler)notFoundHandler targetName:(NSString*)targetName;

/**
*  删除Target缓存
*
*  @return
*/
+ (void)removeTargetsCacheWithTargetName:(NSString*)targetName;
+ (void)removeTargetsCacheWithTargetNames:(NSArray*)targetNames;
+ (void)removeAllTargetsCache;

/**
不定参静态方法调用 （最多支持7个，原因不定参方法传给不定参方法实在没啥好办法。。。。暂时如此）
id result=(__bridge id)zyq_invokeSelectorObjects(@"Class", @"actionWithObj1:obj2:obj3",obj1,obj2,obj3,nil);

c类型转换配合__bridge_transfer __bridge
利用IMP返回值只是指针，不支持C基本类型

@param className 类名
@param selectorName,... 方法名，不定参数
@return 返回值
*/
void * zyq_invokeSelectorObjects(NSString *className,NSString* selectorName,...);
```
最后就是页面路由和方法路由遇到找不到的处理方案了，主要思路就是不crash、好判断，页面路由就判断一下是网页的就跳转url不是就报个提示算了，方法路由return nil吧。。这里仁者见仁智者见智，反正可以自己定制，差不多就讲到这吧。

# 事件链路由
``` Objective-C
/**
响应链传递路由

用于解决多级嵌套UI对象的上级事件响应，省去delegate protocol逐级传递，跨级传递

@param eventName 事件名
@param userInfo 扩展信息
*/
-(void)zyq_routerEventWithName:(NSString *)eventName userInfo:(id)userInfo;
```
这个主要解决多层级UI对象嵌套的时候，事件传递繁琐，通过Event完成对事件定义，一级级传递到响应者的过程在开发中就可以省略了。
只需要如下使用的时候在发起和接受地方写好处理即可！

``` Objective-C
//调用
[self.nextResponder zyq_routerEventWithName:eventName userInfo:userInfo];

//承接
- (void)zyq_routerEventWithName:(NSString *)eventName userInfo:(NSDictionary *)userInfo {
//判断eventName做出对应逻辑
}
```
