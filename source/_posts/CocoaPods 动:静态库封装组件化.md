---
title: CocoaPods 动/静态库封装组件化
date: 2017-08-15 23:07:49
tags:
    - 工程管理
    - IOS
    - 组件化
    - CocoaPods
    - 动/静态库
---
![DingTalk20170818200900.png](/assets/blogImage/3994053-1fd000b267fa2f10.png)
# 动/静态库混用
pods的动静态库混用，相信大多数人一想到就会头皮发麻，体会过的应该都懂，那种无助感。。。。
### 问题
大型项目里来个尝试性swift过渡，首先就是pod加use_frameworks!来支持pod动态库，接着就是一大片的不支持动态库pod提示error，源码上看不dependency依赖静态库pod其实是不会有问题的，如果你的pod全是源码型也不会有任何问题！

ps:其实我感觉这是bug。。。压根就不是啥问题

正常的pod当静态库用的时候vendored_library，vendored_frameworks两个属性搞定一切。。。
### 解决
#### 未来
虽说是未来，但我估计也就只有几个月的事吧，CocoaPods马上就要可以加动静态库标签了！长期关注pods源码的估计不以为然，上个月就定下来1.4.0发布，然而现在是1.3.1。。。。。
那老哥们PR了好久https://github.com/CocoaPods/Core/pull/386
看源码写的用法每个pod是可以定义static_framework:true属性,这样其实对简单的小项目半点作用都没有，但大型项目简直是救世的存在！承接上边的问题，对指定pod加静态库标签瞬间解决！
<!-- more -->
#### 现在
接着扯一下现在的解决方案，首先我们针对的就是pod提供framework和.a的情况，核心问题其实是自己怎么建立对framework和.a的支持。
把dependency一些静态库的pod拍平就是现在的解决方法，自己建pod，保证一层支持framework和.a,另外如果实在自己的pod里dependency的静态库pod，这个时候比较好的选择是建立subspec，直接subspec里面封装对静态库的支持。
这里支持的时候要分为3类，先放一个友盟分享的例子：
友盟的framework 和.a都是静态库
``` 
Pod::Spec.new do |s|
s.name                  = 'XXXThirdPartSocial'
s.version               = '1.0'
s.summary               = '第三方社交模块'
s.homepage              = 'xxx.xxx.xxx'
s.license               = { :type => 'MIT'}
s.author                = { 'xxx' => 'xxx@163.com' }
s.source                = { :git => 'xxx.xxx.xxx', :tag => "#{s.version}" }
s.platform              = :ios, '7.0'
s.source_files          = ''
s.requires_arc          = true
#s.ios.dependency  	'UMengUShare/Social/ReducedQQ'
#s.ios.dependency  	'UMengUShare/Social/ReducedWeChat'
#s.ios.dependency  	'UMengUShare/Social/ReducedSina'
s.subspec 'XXXThirdPartSocialVendor' do |sss|
sss.source_files            = ''
sss.resource                = 'UMSocialSDK/UMSocialSDKPromptResources.bundle'
sss.ios.vendored_frameworks = 'UMSocialSDK/UMSocialCore.framework','UMSocialSDK/UMSocialNetwork.framework'
sss.ios.vendored_library    = 'SocialLibraries/QQ/libSocialQQ.a','SocialLibraries/Sina/libSocialSina.a','SocialLibraries/WeChat/libSocialWeChat.a'
sss.ios.public_header_files   = 'SocialLibraries/**/*.{h}'
sss.ios.library  = 'sqlite3'
end
end
```
![DingTalk20170817182046.png](/assets/blogImage/3994053-b83bb9df3c448f43.png)
##### 动态库Pods封装.a
对.a封装的时候vendored_library属性对应.a，然后看看依赖啥系统库在library，frameworks里加上，最后就是.h,如果你不想暴露的话public_header_files 里加完就不用管了，如果想要暴露给别人调用，只能source_files里再加一遍.h。
上面例子中XXXThirdPartSocialVendor里的source_files为空，但其实.a里的东西你是可以调用的，原因是友盟在他的framework里的头文件引用了.a的头文件，间接让.a的.h公开,这问题在我看来感觉是个bug。。。
所以不想在source_files里再写一遍的也可以建个.h引用一遍所有.a的头文件，最后source_files写你自己的.h,但这只是保证我到处可以通过引用自己的头文件实现方法调用，并不能单个引用对应.a的头文件
##### 动态库Pods封装静态Framework
对静态的Framework封装的时候可以说是最舒服的了，vendored_frameworks加上去基本就万事大吉了，至于依赖啥系统库加library，frameworks这件事，亲测有的时候并不需要！
##### 动态库Pods封装动态Framework
对于动态的Framework封装，我不说估计大家也基本能猜到吧！这就是最难受的，具体情况具体分析，不同情形下用不同套路，就算不用pod也让你很不爽，这里我拿环信客服SDK来讲！
![DingTalk20170817183749.png](/assets/blogImage/3994053-1e7f9f22cfc89904.png)
不用pod你要手动把这SDK拖到上边Embedded Binaries位置头文件才能引用，这个是苹果现在引用动态Framework的套路。。。好烦！
下面讲一下pod怎么搞，如果单纯framework做pod，首先public_header_files要指定xxx.framework/Headers/{*.h}不然你头文件找不到，其次source_files里看具体编译情况决定加不加xxx.framework/Headers/{*.h}，然后就是比较普通的地方vendored_frameworks指定好完事大吉！source_files这个加了的时候还有一个前提就是Framework内引用全是""不能<>，所以大部分情况source_files不加
另一种混合使用感觉这才是最常见的
![DingTalk20170818125422.png](/assets/blogImage/3994053-f6b8369f75849747.png)
这时候不要指定Framework的public_header_files，写一个自己的头文件引用类，把想公开可以调用的在这里#import <xxx.framework/xxx.h>,只能间接把那些搞出去，起作用的只有vendored_frameworks
#### 动态库Pods封装资源文件的调用
高能预警！超级天坑降临！
当你use_frameworks!这么一下你如果自定义的pod有关于resource或resource_bundle的话应该会发现真正的末日降临了，之前的资源全读不出来了！
![DingTalk20170818141324.png](/assets/blogImage/3994053-79ea56d0ef0438f4.png)
一张图片告诉你发生了什么，pod构建动态库的时候你的资源文件都在Framework里！
现在的选择变成：要么资源文件放外面单独加，要么改代码。。。。就问你坑不坑？
放外面单独加我这就不说了太简单，代码写的话其实也要看本身代码的结构什么样，如果像我举例中的SDK基本没救，没有统一的地方获取NSBundle,也没对bundle名称做统一，更没对UIImage设置加扩展！
下面简单说下调用方法
```Objective-C
NSString * bundleName=@"Frameworks/xxx.framework/xxx.bundle";

NSString * bundlePath = [[[NSBundle mainBundle] resourcePath] stringByAppendingPathComponent: bundleName];

NSBundle *bundle= [NSBundle bundleWithPath:bundlePath];

//本地化（拖入.lproj 文件夹即可）
NSString *localizedString=[bundle  localizedStringForKey:@"localizedStringKey" value:@"" table:nil]

//图片获取
NSString *bundleImageName=[bundleName  stringByAppendingString:@"myIcon"];

UIImage *tmpImage=[UIImage imageNamed: bundleImageName];

```
根据上面代码可以找个单例提供bundleName字段，在模块初始化的时候，先判断xxx.bundle有没有，没有的情况bundleName设置成Frameworks/xxx.framework/xxx.bundle，如果没有bundle而是单纯的资源文件，指定到framework目录里就可以随便用了。

只有这样你的pod才算是支持了CocoaPods 动/静态库,也才算是真正的组件化，而且拆包看.app也会感觉优雅很多，分类明确！
# CocoaPods 组件化常识普及
### 模块开发
在模块开发时可以podfile里指定本地路径，.podspec引入工程内但不要加到target里。这样改podspec方便，而且只是修改文件的话可以随时看到pod的真实效果。
上面说的并不是最好的方式，bug fix可能还不错，真开发一大块没有的东西还是先开发最后写podspec为上策！
![DingTalk20170818182404.png](/assets/blogImage/3994053-4cdcb0ac2b4b4918.png)
```
  pod 'xxxModule',:path => '../xxxModule.podspec'
```
### 指定Spec Repo
在不指定Spec Repo的情况事实是如下面类似写podfile文件,的确也可以更新，但pod内如果有dependency就不行除非你在外部也pod了你dependency的库。。。如此一来你的podfile就比较臃肿看起来，当然的确是可以凑活用的！
```
  pod 'xxxxx',:git => 'ssh://git@git.xxxx.cn/xxxxx.git',branch:'master'
```
其实完全可以在podfile最上面写上
```
source 'ssh://xxx.com/my/Specs.git'
```
你只需要把官网的Specs弄下来传到你自己的服务器上，至于怎么上传.podspec文件，可以看下面：
```
#设置一次我们自己的远程仓库源，类似git remote的概念，REPO_NAME仓库源名，SOURCE_URL仓库源远程地址
pod repo add REPO_NAME SOURCE_URL

#上传.podspec文件到自己的本地命名好的那个仓库源
pod repo push REPO_NAME SPEC_NAME.podspec
```
不过没事的时候要合并一下https://github.com/CocoaPods/Specs，不然你的外部引用可就跟不上时代的潮流了！

弄懂上面那些基本上已经够做好工程管理的组件化了！
