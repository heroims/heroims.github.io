---
title: 'Cocoa(iOS,OSX)安保系统设计实现'
date: 2017-12-21 18:38:50
tags:
	- IOS
	- OSX
	- 安保系统
	- Crash防御
	- Cocoa
---
# 前言
这里主要以iOS和OSX讲讲crash闪退怎么防御。
其中最新的OSX应用本身就有一定闪退防御，但有点类似`@try @catch`在最外层包了一下普通的越界调用空方法都会中断在操作位置不向下执行，如果没有进一步复杂逻辑不会闪退，只是影响后续的操作。

而iOS则没这么好说话了，二话不说直接闪退给你看没有上面的那种机制。

所以才有了设计一个安保系统的意义，来保证最大程度的健壮性，理想的状态就是不crash且能继续正常运行后面的逻辑。

参考了众多网上的资料有了下面的小成果分享出来，这其实只是安保系统最后的一个环节的防御

https://github.com/heroims/SafeObjectProxy

# 安保系统设计
这里我所认为的安保系统应该从代码和规范两个层面看，毕竟想抓到所有的crash情况是一定不可能的，现实中即使处处try catch都没法保证抓到所有crash！
## 代码
- swizzing切面
- 方法防御选型
- 防御成功上报

<!-- more -->

程序内需要的是代码，这个模块是要没有任何侵入性的，所以切面是必须的，其次就是尽量的细化切面颗粒度保证意外情况最小化！

另一点就是切面以后我们对原方法应该采取怎样的防御，这里即可以`try catch`的形式也可以进行逻辑判断形式。
而我的代码里用逻辑判断，更多的考量是针对的函数都偏下层且容易使用时外部恰巧又有各种循环逻辑，那样相较之下`try catch`在不间断的调用性能会有一定影响，所以暂时没用`try catch`作为防御的手段。
从另一角度看其实`try catch`的使用场景有些方法还是比较合适的，首先我们在防御时方法颗粒度已经很细所以抓住异常都会做对应处理不会有内存泄漏或逻辑遗漏，另外无论try还是catch内的方法也不会太多，满足了`try catch的最佳场景，只是个别方法循环利用略过高可能性能没法到达极致仅此而已。

防御完了crash就是上报，我们保护了程序的同时也就意味着有地方写的有问题，由于没crash所以没crash log，这时候就需要在安保模块里加入上报机制，这时候我的做法则是放出一个协议等人去实现，安保模块就专心处理防御的事情，上报到服务端的事情交给专门处理这事的模块，我们只需要在防御成功时告知协议有这么个事情即可。剩下的就是个人看情况如需详细情况直接`[NSThread callStackSymbols]`把栈信息输出一下！
```
//安保模块上报协议
@protocol SafeObjectReportProtocol

@required
/**
 上报防御的crash log
 
 @param log log无法抓到Notification的遗漏注销情况
 */
-(void)reportDefendCrashLog:(NSString*)log;

@end
```
而实现这个协议的只需要对`SafeObjectProxy`做个Category实现一下即可。

还有就是防御的分类开启，这时候枚举就要用位运算的形式，这样才能兼容多种模式并存如下只开启Array和String的防御
```
[SafeObjectProxy startSafeObjectProxyWithType: SafeObjectProxyType_Array| SafeObjectProxyType_String]
```

## 规范
另一个安保模块的组成则应该是对代码规范的制定与校验，这就需要clang来做了，不是这里主要讲的，相当于多了一种`Build Options`的`Compiler for C/C++/Objective-C`属性的选择，用我们开发的Xcode校验插件，检查代码语法上的问题直接报错，这样从源头来规范化编码。

# Crash分类及防御实现
- Unrecognized Selector(找不到方法)
- UI Refresh Not In Main Thread(UI刷新不在主线程)
- Input Parm Abnormal(入参异常)
- Dangling Pointer(野指针)
- Abnormal Matching(异常配对)
- Thread Conflict(线程冲突)

想要防御crash，首先要做的就是了解都有哪些情况会产生crash,上边就是笔者总结的几种最常见的情况，不全的话希望有人留言补足，毕竟crash的防御真正有发言权开发这种模块的估计只有大公司开发app的，不然用户量不够没样本采集，没法了解坑爹的情况！

而上面列的6种常见crash，真正能广域控制得了的恐怕也只有一半不到！下面就一一讲解一下,Hook切面就是主要的手段！

## Unrecognized Selector(找不到方法)
这个找不到方法算是比较好办的。。。也算是比较常见的好查的，另外处理ok了null对象调用的问题也会随之解决
可选的方法有两种
Hook这两个方法
`- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector`
`- (void)forwardInvocation:(NSInvocation *)anInvocation`
或Hook这一个方法
`-(id)forwardingTargetForSelector:(SEL)aSelector`

核心思想就是在找不到方法之前创建方法确保继续执行不挂，为了尽量不多余的创建方法，集中的把创建打到统一的地方。

前者需要在`methodSignatureForSelector`执行前在新的target里创建没有的方法，然后用它调用`methodSignatureForSelector`返回，而这里的target当然要单例弄出来省的以后来回创建。然后在`forwardInvocation`里用他来调用`invokeWithTarget`指到我们新的target上。

后者也就是我用的方法，之所以用它主要是一个方法 就ok！而我们还要兼顾静态方法和实例方法去分别hook才能防住这两种，而前者也要hook的方法更多。。。。
而这里只需要切`forwardingTargetForSelector`方法，静态方法返回class，动态方法返回target，当然返回之前我们要添加上不存在的方法，值得注意的是OSX上一个神奇的问题，我在判断是否系统有这个方法的时候第一次居然`respondsToSelector`返回false而`methodSignatureForSelector`有数据，第二次校验是`methodSignatureForSelector`才为空，而iOS上则没这问题第一次校验就是对的！
## UI Refresh Not In Main Thread(UI刷新不在主线程)
刷新UI不在主线程的情况这里只是针对UIView和NSView的3个方法做切面线程判断。分别是`setNeedsLayout`,`setNeedsDisplay`,`setNeedsDisplayInRect`，执行之前看是不是在主线程，不在的话就切到主线程执行，但很明显这3个方法肯定覆盖不全，而且就算覆盖全了每次都判断一下也是性能浪费，所以这里各自斟酌处理吧，这类情况暂时没想到其他好的处理方式！但好在算是有这么个可控方案！
## Input Parm Abnormal(入参异常)
入参异常这是一大类，防御的方法也相对比较通俗易懂，也是最容易查最容易出现的。
### 常用类型入参异常
常见类包括String，Array，Dictionary，URL，FileManager等这些类空值初始化，越界取值，空赋值等，基本看crash log统计依次切面对应方法在执行前判断一下就ok。如`objectAtIndex`,`objectAtIndexedSubscript`,`removeObjectAtIndex`,`fileURLWithPath`,`initWithAttributedString`,`substringFromIndex`,`substringToIndex`等等。唯一需要注意的就是这些要切面的类名可是五花八门而且更iOS版本有很大关系，所以这个就是靠crash log积累了解有哪些坑。当然代码写的好就用不到了！`__NSSingleObjectArrayI`这个就是最近在iOS11上新发现的报错数组类，当然也可能是最近我司有人写出了这个相关的bug......
常见的需要注意的hook的类有以下
`objc_getClass("__NSPlaceholderArray")`
`objc_getClass("__NSSingleObjectArrayI")`
`objc_getClass("__NSArrayI")`
`objc_getClass("__NSArrayM")`
`objc_getClass("__NSPlaceholderDictionary")`
`objc_getClass("__NSDictionaryI")`
`objc_getClass("__NSDictionaryM")`
`objc_getClass("NSConcreteAttributedString")`
`objc_getClass("NSConcreteMutableAttributedString")`
`objc_getClass("__NSCFConstantString")`
`objc_getClass("NSTaggedPointerString")`
`objc_getClass("__NSCFString")`
`objc_getClass("NSPlaceholderMutableString")`
具体有哪些方法需要切面还是看源码吧，这部分是没什么难点的。

另外我的防御里面没对NSCache做，可能以后会随便加点，因为缓存相关的模块我的建议是自己封装缓存模块或用第三方，那样对于上层使用者来说已经是安全的了！各种异常处理在缓存模块里就应该有封装。

### KVC Crash
KVC归根结底也算这类入参异常，一共切面3个地方就够防御了！
`-(void)setValue:(id)value forKey:(NSString *)key`,
`-(void)setValue:(id)value forKeyPath:(NSString *)keyPath`
空值防御上面2个方法
`-(void)setValue:(id)value forUndefinedKey:(NSString *)key`
上面这个就是没有的属性做赋值操作时走的回调，如果用到我的[SafeObjectProxy](https://github.com/heroims/SafeObjectProxy)要自定义各个类不同的处理是可以不开启`UndefinedKey`防御的！

## Dangling Pointer(野指针)
这个种Crash堪称经典！就是那个最难排查的，而这里我们能做的防御事情也十分有限！
具体定位看看腾讯这几篇很有帮助！
 [如何定位Obj-C野指针随机Crash(一)](https://dev.qq.com/topic/59141e56ca95d00d727ba750)
 [如何定位Obj-C野指针随机Crash(二)](https://dev.qq.com/topic/59142d61ca95d00d727ba752)
 [如何定位Obj-C野指针随机Crash(三)](https://dev.qq.com/topic/5915134b75d11c055ca7fca0)
我们只能去对已知的出现野指针的类进行防御，找到crash的野指针开启Zombie Objects，加上Zombies工具，然后想办法不断提高复现率还是可以的定位到的。
我们的防御则是hook系统dealloc，判断需要做处理的类不走系统delloc而是走`objc_desctructInstance `释放实例内部所持有属性的引用和关联对象,保证对象最小化。紧接着就需要来波`isa swizzling`了，因为通常野指针伴随着的还有就是调用没有的方法，或者由于调用的这个时机是不正常的，各种数据的安全性都没了保证，所以dealloc后解除所有持有，再把原来的isa指向一个其他的类，而这个类能把所有的调用方法指向一个空方法这样就起到了防御的作用。

能干这事的也只有NSProxy了，利用协议实现`methodSignatureForSelector `，`forwardInvocation `方法，统一打到之前处理找不到方法自动创建的类中，也就是在NSProxy内实现上面`Unrecognized Selector`的防御，这样所有对于野指针的调用就都是空了！
正因为上面的原因一旦开启了这个防御，真正释放的时机就还是有的，如果在野指针出现前触发了真正释放的逻辑，crash就还是会有的！
我在[SafeObjectProxy](https://github.com/heroims/SafeObjectProxy)里只是用野指针个数控制做真正释放，回头可能会封装个block方便复杂情况的判断。
## Abnormal Matching(异常配对)
这一类算是不建议做防御的！成对的方法处理异常像KVO，NSTimer，NSNotification都算，需要注册和注销。
这种情况我的建议是统一封装独立模块调用统一的方法，让人不需要关心注册和注销，主要写逻辑处理。从功能实现上做严格限制，这样让人考虑的就是怎么样把一个场景融入到封装的方法中，而不是随意的写！
下面说下原因，由于注册和注销是分离写的 ，所以使用场景，解决问题的方法都会有着非常灵活的操作，这其实很可怕，先用KVO做一个举例顺便说一下这类防御如果真要做一般的做法是怎么做。
### KVO
KVO这种crash如果要防御其实只能防御下面3种情况：
1.观察者或被观察者已经不存在了
2.取消和添加的次数不匹配
3.没写监听回调`observeValueForKeyPath:ofObject:change:context:`

而这3种情况我们来认真思考下开发的阶段是不是貌似都会第一时间就被发现！而且如果是没经验的程序员写KVO我们是不是都不敢用，会再三审查，而有经验的又不会犯上面的错。。。。
如果对上面的情况防御也很复杂，而且我尝试并且用过很多第三方，都在我司稍微有点复杂的项目上挂了，不仅没能防御crash还造了crash，这种成对逻辑的灵活性非常高，你没法知道系统内部人家怎么用着玩的！
说一下防御上面的情况首先是吧，切面add、removeObserve是一定的，还要在所有的类里对再加一个对象，这个对象主要负责管理KVO下面就叫KVOController吧，让所有的观察者都成为了被观察者的一个属性，用map记录原来的观察者和keyPath等信息,这样添加或移除观察者就能判断是不是成对出现的，另外KVOController在dealloc时也可以通过map依次移除监听，而由于所有的监听回调其实都是由KVOController的`observeValueForKeyPath:ofObject:change:context:`通过`[originObserver observeValueForKeyPath:keyPath ofObject:object change:change context:context]`传递出去的自然没写监听回调的情况也可以判断了，但也是能解决那3个情况！

真正KVO产生的恐怖的crash是移除时机不和观察者或被观察者销毁有关系，而是跟我们的逻辑有关，一旦没在合适时机移除导致的crash排查起来超级费劲！还有你在监听回调里处理逻辑有没有线程安全问题，这些才是我们在上线前容易漏，排查又不好排查的！

安保系统则是要保护上线后能正常运作，然而就像我这里说的KVO，如果不在编码期间就做严格规范，上线后出的问题也是根本无从防御的！

然后再来说说怎么限制我们的自由发挥，KVOController刚才说到的这里需要的是把它变形，把回调用block放出来，另外就是让它有单例模式和普通的实例模式，只有创建对象、关联监听和逻辑处理，一个KVOController可以是全局或属于一个对象，相当于可视化了KVO的生效周期，一目了然，这里让特殊逻辑适应我们的规范才是正确的安保思路。包括NSTimer在内也也是如此可以搞个TimerController不过封装最好也别用NSTimer精度不高，反正要封装不如直接gcd，与其要手动保持成对不如我们就把逻辑封装好，让使用者忘掉成对的概念！但在开放的今天完全可以GitHub搜一波找些封装好的自己再简单包装下，然后让团队遵循规范开发即可。。。

KVO:[KVOController](https://github.com/facebook/KVOController)比较推荐的一个KVO管理

### NSTimer
NSTimer比较特殊，有些时候偏偏不该成对使用，它的成对的逻辑其实是跟自己的生命周期有关，毕竟生命周期结束时要去成对的停掉timer才能释放，另一点就是NSTimer精确度并不高！但它封装出来给人用的方法是ok的正是有单例模式和实例模式两种使用。所以我的建议当然是自己把gcd的timer封装一下，另外把target这个概念变为weak持有，这样我们自己封装的timer就可以dealloc的时候停掉timer释放了，按照系统NSTimer封装方法即可。这样至少能保证timer指定的target释放时timer能停掉不会因为跑了其他不安全的逻辑挂掉。其他可能挂掉的情况应该比较少。。。

Timer:[MSWeakTimer](https://github.com/mindsnacks/MSWeakTimer)比较推荐的一个计时器封装方法就是我上面讲的那种
### NSNotification
这个虽然也是成对使用，单比上面的几个要安全一些，因为使用它有`[[NSNotificationCenter defaultCenter] removeObserver:self]`多次调用或没`addObserver`都不会挂，所以可以全局搞一下，我在[SafeObjectProxy](https://github.com/heroims/SafeObjectProxy)里面就只是对所有`NSObject`对象添加了个属性做标识，然后hook一下`NSNotificationCenter`的`-(void)addObserver:(id)observer selector:(SEL)aSelector name:(NSNotificationName)aName object:(id)anObject`方法，只要observer是`NSObject`对象我就标识一下，然后切所有`NSObject`的`dealloc`只要标识了的统一执行`[[NSNotificationCenter defaultCenter] removeObserver:self]`，反正多执行了也没问题用的放心！

但只要是成对的，就有另一个问题，万一真正需要注销的地方是跟逻辑有关，那你对象销毁时注销早就晚了，就像上面KVO中提到的我们做的这层crash防御其实犯错率并不高能及时发现，而及时发现不了的只能是通过编码规范或者人员分级禁用来解决。
## Thread Conflict(线程冲突)
基本无解的问题，出现以后瞬间懵逼，典型例子就是死锁，异步调用同一对象导致不安全，基本没有防御手段，排查也只能靠多加log不断复现，然后猜。。。。
但一般只要代码按照正常的规范写也不会那么容易遇到这问题，但线程冲突理论上只要保证UI操作都在主线程，其他都gcd不在主线程上，然后部分需要线程安全的gcd信号量做锁就可以，但不会有人这样写代码，性能和效率那么搞是都要废的，现在都恨不得你马上出活那有空那样，这类就可以完全不考虑防御的事了！


