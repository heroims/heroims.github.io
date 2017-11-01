---
title: Xcode 配置多种scheme完美使用（配合Pods）
date: 2017-11-01 20:21:10
tags:
    - IOS
    - Scheme
    - 预编译宏
    - Configurations
    - Preprocessor Macros
    - User-Defined-Setting
    - CocoaPods
---

# 前言
这里说到scheme其实配置不难，但真正应用到大项目中会发现一个神奇的问题，调试的时候自己自定义的scheme变量值都是nil，即使配置好也那样，主要场景就是工程内的其他工程，所以你的配置其实是要应用到所有子工程下的，是不是瞬间压力山大，，，，，本文最后就讲讲结合pod后轻松解决的办法，开头还是由浅入深，这样受众多点，文章也不至于太单调，就从配置开始一路讲到调试使用。
# 配置 Configurations
这一步主要是创建我们的编译配置项，比如添加备机,测试环境的调试和发布项，下面是添加了测试环境

![屏幕快照 2017-10-31 下午7.10.33.png](/assets/blogImage/3994053-36feba23b8a9e57d.png)
这里环境分离时最好也是分Debug和Release,我添加了SchemeAppTest_Debug和SchemeAppTest_Release相当于测试环境下的Debug和Release还有每次创建新的一定要基于Debug或Release不然出现啥配置不对真的是会排查到疯！
# 配置 Preprocessor Macros
![屏幕快照 2017-10-31 下午7.28.20.png](/assets/blogImage/3994053-0cbf93e2a6e0e7b9.png)
有了配置项，现在需要添加预编译宏，这样我们的代码里就可以用它来判断我们正在编译的是哪个环境了。
上图我分别对SchemeAppTest_Debug和SchemeAppTest_Release配置了`SCHEMETESTDEBUG=2`和`SCHEMETESTRELEASE=3`，内部其实写什么都可以只不过用的时候就要用你写的哪个宏，这里的写法只是效仿系统，完全可以把SchemeAppTest_Debug定为`ABC=1000`,也没问题，只不过用的时候就如下就可以了，ABC就代表了SchemeAppTest_Debug
```
#ifdef ABC
#else
#endif
```
<!-- more -->
# 配置 Scheme
首先要创建scheme这里我创建的SchemeApp-Test就是测试环境用的
![屏幕快照 2017-10-31 下午7.42.15.png](/assets/blogImage/3994053-87e9c9897c77efce.png)

![屏幕快照 2017-10-31 下午7.43.21.png](/assets/blogImage/3994053-3112177a3c76dafa.png)
添加完成就需要配置对应的Configurations，我们是基于原项目创建的，所以简单改下就可以。
![
![Uploading 屏幕快照 2017-10-31 下午7.47.46_551095.png . . .]
](/assets/blogImage/3994053-6076902ef0296120.png)
![屏幕快照 2017-10-31 下午7.47.46.png](/assets/blogImage/3994053-5739deca1545b24e.png)
其实常用的就只有Run，配置一下就基本满足大部分需求，做完这些再把当前的scheme设置成shared就算是完整的配置完了scheme了
![
![Uploading 屏幕快照 2017-10-31 下午7.58.22_122575.png . . .]
](/assets/blogImage/3994053-76c183e70ef95e13)

![屏幕快照 2017-10-31 下午7.59.00.png](/assets/blogImage/3994053-c86836b8b7d2d84b)

# 使用
看样子好像一切都已经完美搞定剩下的就是使用了，其实正题才刚刚开始。。。。
首先是最常用的用法就是直接使用宏来框自己代码，这个上面其实已经有提到，这里只是写个正式点的如下,定义了BaseHost不同情况下的url
```
#ifdef SCHEMETESTDEBUG
#define BaseHost @"http://xxx.xxx.xxx/test"
#define CanUseCustomHost YES
#elif SCHEMETESTRELEASE
#define BaseHost @"http://xxx.xxx.xxx/test"
#define CanUseCustomHost NO
#else
#define BaseHost @"http://xxx.xxx.xxx"
#define CanUseCustomHost NO
#endif
```
还可以配置不同AppIcon和LaunchImage对应不同环境如下图
![屏幕快照 2017-10-31 下午8.16.10.png](/assets/blogImage/3994053-17ea70096ec3e073)
另外User-Defined-Setting也是还算常用的一个配置,如下图添加了一个APPCustomName，这个用的时候写成`$(APPCustomName)`

![屏幕快照 2017-11-01 上午11.55.32.png](/assets/blogImage/3994053-b39c7f556d923efc)

![屏幕快照 2017-11-01 下午12.06.32.png](/assets/blogImage/3994053-92d4b200d7f69e69)
另外就是plist里所有值都可以按上面方法做到环境分离，写法保持`$(xxxx)`即可，这样配合Build Settings也可以把Bundle Identifier用这种方法做环境分离打出不同环境的包，这样不同环境的包就可以在手机上并存了。
# 调试
重头戏现在才开始，上边的搞完，就能打包看效果了，但是一旦你的项目里包含了其他项目或用了pod，瞬间你就会发现调试的时候NSLog输出变量都在断点的话则什么都没有!如图，这是pod方式用AF，到AF里打断点外部传的变量都是nil。。。。
![屏幕快照 2017-11-01 下午3.46.15.png](/assets/blogImage/3994053-ea973b927dfa3697)


![屏幕快照 2017-11-01 下午3.46.37.png](/assets/blogImage/3994053-bf61aba6d19b56e6)

直接引用一个工程项目也会有此情况，当然你要引用的是第三方库没法调试就不用考虑了，这里原因其实很简单,看下图

![屏幕快照 2017-11-01 下午4.01.51.png](/assets/blogImage/3994053-defff1bd06fc295d)

你需要确保所有引进来的工程`Optimization Level`这个属性想要支持调试必须是'None[-O0]'，和Debug一样的设计，其次如果组件化的设计，你的预编译宏的值也要在你Module的工程里有设置，换句话说现在引进来的其他工程你的预编译宏在他们里面是没有生效的！
![屏幕快照 2017-11-01 下午4.07.54.png](/assets/blogImage/3994053-50a770f7ab269284)

所以你还需要把每个工程里都设置一遍`Optimization Leve`和`Preprocessor Macros`，手动设置就不说了,还有这里这里如果真的时间到大型项目里很可能要设置的不止2个配置，这里只是举了最容易发现的两个问题，同理解决即可。如果用pod，我们所希望的当然是`pod update`完就帮我们设置好！

下面就放一波代码，以当前情况为例
```
target 'SchemeApp' do
# Uncomment the next line if you're using Swift or would like to use dynamic frameworks
# use_frameworks!

# Pods for SchemeApp
pod 'AFNetworking'
target 'SchemeAppUITests' do
inherit! :search_paths
# Pods for testing
end

end

post_install do |installer_representation|

installer_representation.pods_project.targets.each do |target|
target.build_configurations.each do |config|
#去警告
config.build_settings['GCC_WARN_INHIBIT_ALL_WARNINGS'] = 'YES'
#判断scheme
if config.name.include?("SchemeAppTest_Release")
#添加scheme对应的预编译宏
config.build_settings['GCC_PREPROCESSOR_DEFINITIONS'] ||= ['$(inherited)']
config.build_settings['GCC_PREPROCESSOR_DEFINITIONS'] << 'SCHEMEAPPTESTRELEASE=3'
end
if config.name.include?("SchemeAppTest_Debug")
config.build_settings['GCC_PREPROCESSOR_DEFINITIONS'] ||= ['$(inherited)']
config.build_settings['GCC_PREPROCESSOR_DEFINITIONS'] << 'SCHEMEAPPTESTDEBUG=2'
#指定scheme的调试模式可见变量
config.build_settings['GCC_OPTIMIZATION_LEVEL'] = '0'
#某些情况由于编译器不支持无法debug（可选）
config.build_settings['ONLY_ACTIVE_ARCH'] = 'YES'
end
end
end
end

```
利用好podfile文件,就这样一次性解决,把所有pod都配置了一遍`Build Settings`的设置，至此scheme才算完美搞完，利用上面的思路，即使项目再神奇应该也能做到完美的配置scheme。
