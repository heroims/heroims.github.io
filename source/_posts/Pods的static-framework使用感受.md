---
title: Pods的static_framework使用感受
date: 2017-09-29 16:39:56
tags:
    - IOS
    - CocoaPods
    - 动态库
    - Framework
    - 静态库
---
# 前言
CocoaPods终于出1.4.0.beta.1了，终于能用用static_framework这个属性了，拯救世界的属性啊！
### 使用场景
封装pod依赖另一个有.a或Framework的pod，然后用`use_frameworks!`来动态库模式pod。那情况原来只能把那有.a或Framework的pod直接代码都放自己pod里，不能dependency，当然项目没用任何Swift其实就没事因为用不到`use_frameworks!`。还有就是Pod里用了资源文件的，因为用了资源文件和`use_frameworks!`，最终生成资源可都在.app的Frameworks/xxxxx.framework/里，如果第三方代码处理了这种情况还好，不过现在看几乎没有几个做这个处理的，都没想过动态库带来的改变也是醉了，资源都会找不到！使用静态库模式则会拍平就没这问题了。

# 用法
下面来个例子，封装统计模块需要依赖Google统计，Fabric crash统计，现在有了这属性直接如下就完成了依赖关系
```
#Google 统计
s.ios.dependency      'Google/Analytics'
#Fabric crash统计
s.ios.dependency      'Fabric'
s.ios.dependency      'Crashlytics'
#指定Pod为静态库模式
s.static_framework      = true
```

### 注意事项
- 不能在当前pod里使用`subpecs`
- Framework受限头文件引用模式

<!-- more -->
这里举依赖Google的例子主要是人家`podspec`写的好，还有就是`static_framework`属性并不是万能的！遇到环信，友盟分享这类就废，还有其他情况也不行。



下面说说这类情况，第一个`static_framework`不能放到`subpecs`里也不能有`subpecs`，这个倒是不难解决，可以为这个单建一个pod作为依赖的中间件，这样要创建的pod就可以不使用`static_framework`属性了依赖使用了这个属性里面只有dependency的中间件。
第二种就是没救的，典型范例环信和友盟分享，这俩都是对Pod里的Framework认知不足，不过也有部分原因在OC没正经命名空间。
没问题可用`static_framework`的Framework的`podspec`写法如下
```
s.ios.source_files        = 'xxxx.framework/Headers/*.h'
s.ios.public_header_files = 'xxxx.framework/Headers/*.h'
s.ios.vendored_frameworks = 'xxxx.framework'
```
而且内部对xxxx.framework的文件引用只能`#import "xxx.h"`不能`#import <xxxx/xxx.h>`，否则头文件找不到，静态库模式这情况没办法，还有IOS本来Framework就都不是正经的动态库有这问题并不奇怪，同样也因此没法通过Framework隔离同名文件了，没有命名空间概念隔离的情况再现，这就是之前说的OC的一部分原因，所以才会很多都写`#import <xxxx/xxx.h>`感觉踏实。所以现在就尴尬了，国内很多支持pod的sdk是Framework都是直接`s.ios.vendored_frameworks = 'xxxx.framework'`，所以其实这些sdk其实真的该考虑pod的Framework单独处理，给人扔项目里直接用的保持`#import <xxxx/xxx.h>`风格写法没问题，pod很多时候就是为了管理各种依赖关系，至于是动态库还是静态库编译都是看项目本身需要，封装pod的时候应该必须保持`#import "xxx.h"`写法才ok，另外.a在pod里可能更加讨喜吧。
