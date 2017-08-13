---
title: IOS百行代码全局语言本地化/国际化
date: 2017-02-15 02:09:05
tags:
---

# IOS百行代码全局语言本地化/国际化
废话不多说先把基本配置搞定图文一起来

![3618569F-C518-4582-8580-9DC73F1E8B2E.png](http://upload-images.jianshu.io/upload_images/3994053-006976bd750f8324.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里点个“+”直接就o了
剩下的就是创建strings文件也就是语言映射文件

![32E018FA-1E4C-4127-BD87-0AE0B0B1641D.png](http://upload-images.jianshu.io/upload_images/3994053-6172f40aabb17548.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

划线的就是创建时候该点击的InfoPlist.strings就是定义CFBundleDisplayName来确定app的名字
Localizable.strings则是正经的语言映射表了
xib，storyboard和image也是一个套路看划线那点击就ok了
![6DCE952A-4868-4652-AAB5-59C36DF129D5.png](http://upload-images.jianshu.io/upload_images/3994053-f2f6768c6d73d371.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
# 百行代码全局Hook完成本地化
废话不多说先放代码在解释
https://github.com/heroims/LocalizedEngine
<!-- more -->
```Objective-C
#import "LocalizedEngine.h"
#import <objc/runtime.h>
#import <UIKit/UIKit.h>

#define LocalizedString(string) NSLocalizedString(string, nil)

@interface NSObject (LocalizedEngineSwizzling)

@end
@implementation NSObject (LocalizedEngineSwizzling)

+ (BOOL)les_swizzleMethod:(SEL)origSel withMethod:(SEL)altSel {
Method origMethod = class_getInstanceMethod(self, origSel);
Method altMethod = class_getInstanceMethod(self, altSel);
if (!origMethod || !altMethod) {
return NO;
}
class_addMethod(self,
origSel,
class_getMethodImplementation(self, origSel),
method_getTypeEncoding(origMethod));
class_addMethod(self,
altSel,
class_getMethodImplementation(self, altSel),
method_getTypeEncoding(altMethod));
method_exchangeImplementations(class_getInstanceMethod(self, origSel),
class_getInstanceMethod(self, altSel));
return YES;
}

+ (BOOL)les_swizzleClassMethod:(SEL)origSel withMethod:(SEL)altSel {
return [object_getClass((id)self) les_swizzleMethod:origSel withMethod:altSel];
}

@end

@interface NSString (LocalizedEngine)

@end

@implementation NSString(LocalizedEngine)

-(CGRect)le_boundingRectWithSize:(CGSize)size options:(NSStringDrawingOptions)options attributes:(nullable NSDictionary<NSString *, id> *)attributes context:(nullable NSStringDrawingContext *)context{
return [LocalizedString(self) le_boundingRectWithSize:size options:options attributes:attributes context:context];
}

- (CGSize)le_sizeWithAttributes:(nullable NSDictionary<NSString *, id> *)attrs{
return [LocalizedString(self) le_sizeWithAttributes:attrs];
}
@end

@interface UILabel (LocalizedEngine)

@end

@implementation UILabel(LocalizedEngine)

-(void)le_setText:(NSString *)text{
[self le_setText:LocalizedString(text)];
}

@end

@interface UITabBarItem (LocalizedEngine)

@end
@implementation UITabBarItem (LocalizedEngine)

-(void)le_setTitle:(NSString *)title{
[self le_setTitle:LocalizedString(title)];
}

@end

@interface UIViewController (LocalizedEngine)

@end
@implementation UIViewController (LocalizedEngine)

-(void)le_setTitle:(NSString *)title{
[self le_setTitle:LocalizedString(title)];
}

@end

@interface UIButton (LocalizedEngine)

@end
@implementation UIButton (LocalizedEngine)

-(void)le_setTitle:(NSString *)title forState:(UIControlState)state{
[self le_setTitle:LocalizedString(title) forState:state];
}

@end

@implementation LocalizedEngine

+(void)startEngine{
[[UILabel class] les_swizzleMethod:@selector(setText:) withMethod:@selector(le_setText:)];
[[UITabBarItem class] les_swizzleMethod:@selector(setTitle:) withMethod:@selector(le_setTitle:)];
[[UIViewController class] les_swizzleMethod:@selector(setTitle:) withMethod:@selector(le_setTitle:)];
[[UIButton class] les_swizzleMethod:@selector(setTitle:forState:) withMethod:@selector(le_setTitle:forState:)];
[[NSString class] les_swizzleMethod:@selector(boundingRectWithSize:options:attributes:context:) withMethod:@selector(le_boundingRectWithSize:options:attributes:context:)];
[[NSString class] les_swizzleMethod:@selector(sizeWithAttributes:) withMethod:@selector(le_sizeWithAttributes:)];


}

@end
```

调用只需要一句话
[LocalizedEngine startEngine];
其实只对UILabel做一次全局的文本就都被替换了，另一方面就是NSString对可变宽高的处理，这两个就已经解决了大部分需求，只不过后来发现系统控件在传递文本时做了布局的自适应，所以就有了UITabBarItem UIButton UIViewController的扩展，
根据自己APP情况适当调整即可，包括NSAttributedString 细化出来这些估计也不会超过200行，上边放出的是初版，.m总共100行冒头，轻轻松松省去慢慢找位置加，有人找你做本地化，从此只需要做好string对照表即可，让人按格式给你。
这只是利用AOP这种面向切面开发的思想解决问题的实例，同样的道理，风格扩展，日志，crash防御都能很好的独立且省事的完成。
后续还将放一篇AOPLogger的讲讲
