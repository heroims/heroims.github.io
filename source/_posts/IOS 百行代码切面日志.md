---
title: IOS 百行代码切面日志
date: 2017-03-22 02:09:05
tags:
    - IOS
    - 百行代码系列
    - 切面
    - AOP
    - 日志
---

# AOPLogger
### 切面日志作用
说到日志，切面的实现最大的好处也就是分离出来，单独开发，包括埋点，记录输出log都可以在不影响项目内逻辑的情况下完成，形成完全的一个独立模块。
#### 这里的切面日志，只是作为分离业务而用，所以它的作用只是记录，至于上传删除，这些逻辑同样可以分离出来单独写，存的过程解决了渗透入工程内，上传删除包括分级这些逻辑其实本身就很独立。


在实现上用了Aspects这个库，主要就是hook方法，用它主要是对一些例外情况处理不错，自己写简单实现其实就只写写方法交换剩下的就不管了。。。

下面说一下.h
```Objective-C
#import <Foundation/Foundation.h>

extern NSString * const AOPLoggerMethod;//要统计的日志方法Key
extern NSString * const AOPLoggerLogInfo;//要统计的日志信息Key
extern NSString * const AOPLoggerPositionAfter;//方法执行后统计日志
extern NSString * const AOPLoggerPositionBefore;//方法执行前统计日志
extern NSString * const AOPLoggerPositionType;//执行日志统计的类型Key

@protocol AOPLoggerGetConfigInfoProtocol <NSObject>

@required

/**
创建类扩展如果使用此协议必须实现此方法
此方法返回统计的配置信息，可以从网络取也可以从本地取
@return 统计配置字典
*/
-(NSDictionary*)al_getConfigInfo;

@end
@protocol AOPLoggerBLLProtocol <NSObject>

@required

/**
创建类扩展如果使用此协议必须实现此方法
此方法主要来处理切面方法后的log信息处理可以存本地也可以使用其他任意第三方输出
@param log 配置文件里定义的AOPLoggerLogInfo信息
@param originAOP AspectInfo的方法信息，第三方库Aspect返回的切面方法的所有信息
*/
-(void)al_logger:(id)log originAOP:(id)originAOP;

@end

@interface AOPLogger : NSObject


/**
开始读取日志Plist配置文件
*/
+(void)startAOPLoggerWithPlist;

/**
统计日志的调用方法
（如果不想增加开机时间可以采取每个模块创建一个日志统计类适时调用，在该类里提供一个初始化方法，内部调用此即可）
@param classString 类名
@param methodString 方法名
@param log 相当于AOPLoggerLogInfo信息
*/
+(void)AOPLoggerWithClassString:(NSString*)classString methodString:(NSString*)methodString log:(id)log;

/**
统计日志的调用方法
（如果不想增加开机时间可以采取每个模块创建一个日志统计类适时调用，在该类里提供一个初始化方法，内部调用此即可）
@param classString 类名
@param methodString 方法名
@param log 相当于AOPLoggerLogInfo信息
@param logPosition 日志统计时位置，可放在方法运行前或运行后（默认运行后执行日志统计）
*/
+(void)AOPLoggerWithClassString:(NSString *)classString methodString:(NSString *)methodString log:(id)log logPosition:(NSString*)logPosition;

@end
```
<!-- more -->
这里提供了两种方式一种直接读取自定义的Plist，一种就是调用类方法，而即使调用类方法，也是单独建一个类，某个模块的日志类负责记录，传入类名方法名统计信息即可，而plist得形式在初期还好，后期统计曾多可就真的太扯了毕竟要在初始化的时候加载遍历执行
![QQ20170306-012628.png](/assets/blogImage/3994053-46f6bef1c87511f6.png)
# 定制扩展
由于每个项目想要做的事情或逻辑都会有不同，这里就可以根据我的协议实现对应的方法，来完成自己的业务需求
如下：
```Objective-C
import "AOPLogger+Custom.h"
#import "Aspects.h"
#import <objc/runtime.h>

@implementation AOPLogger (Custom)

-(void)al_logger:(id)log originAOP:(id)originAOP{
if ([log isKindOfClass:[NSString class]]) {
NSLog(@"event:%@",log);
}
if ([log isKindOfClass:[NSDictionary class]]) {
NSLog(@"eventName:%@\neventLabel:%@\neventTime:%@",log[@"EventName"],log[@"EventLabel"],[log[@"EventTime"] boolValue]?[NSDate date]:@"不用获取");
}
if (originAOP&&[originAOP conformsToProtocol:objc_getProtocol("AspectInfo")]) {
id<AspectInfo> aspectInfo=originAOP;
NSLog(@"originClass:%@\noriginSel:%@",NSStringFromClass([aspectInfo.originalInvocation.target class]),NSStringFromSelector(aspectInfo.originalInvocation.selector));

for (NSInteger i=0; i<aspectInfo.arguments.count; i++) {
NSLog(@"argument:%@",aspectInfo.arguments[i]);
}
}
}

@end
```
这里只是做了简单的NSLog操作，而当某个业务模块下需要不同处理的时候，不妨就直接hook这个方法添加逻辑，用法完全靠个人想像吧，其实用起来经常能处理很多神奇的逻辑，里面的originAOP如果用过Aspects这个库的话可能会很快明白为什么要选这个库不是自己写了，因为它的这块封装可以让我拿到对应方法执行完后返回的值，当然你得指定这个log是在原方法执行完成后执行，同理如果放在执行前执行log我们通过这个对象还可以拿到方法传递的参数。

再看.m的源码
```Objective-C
#import "AOPLogger.h"
#import "Aspects.h"
#import <objc/runtime.h>

NSString * const AOPLoggerMethod=@"AOPLoggerMethod";
NSString * const AOPLoggerLogInfo=@"AOPLoggerLogInfo";
NSString * const AOPLoggerPositionAfter=@"AOPLoggerPositionAfter";
NSString * const AOPLoggerPositionBefore=@"AOPLoggerPositionBefore";
NSString * const AOPLoggerPositionType=@"AOPLoggerPositionType";

@implementation AOPLogger

+ (AOPLogger *)sharedAOPLogger {
static AOPLogger *sharedAOPLogger = nil;
static dispatch_once_t onceToken;
dispatch_once(&onceToken, ^{
sharedAOPLogger = [[self alloc] init];
});
return sharedAOPLogger;
}

+(void)startAOPLoggerWithPlist{
NSDictionary *loggerConfigInfo=nil;
if ([[AOPLogger sharedAOPLogger] conformsToProtocol:objc_getProtocol("AOPLoggerGetConfigInfoProtocol")]) {
loggerConfigInfo=[(AOPLogger<AOPLoggerGetConfigInfoProtocol>*)[AOPLogger sharedAOPLogger] al_getConfigInfo];
}
else{
loggerConfigInfo=[NSDictionary dictionaryWithContentsOfFile:[[NSBundle mainBundle] pathForResource:@"AOPLoggerConfig" ofType:@"plist"]];
}

for (NSString *className in loggerConfigInfo) {
for (NSDictionary *eventInfo in loggerConfigInfo[className]) {
Class clazz = NSClassFromString(className);
SEL selector = NSSelectorFromString(eventInfo[AOPLoggerMethod]);
AspectOptions positionOptions=AspectPositionAfter;
if ([loggerConfigInfo[AOPLoggerPositionType] isEqualToString:AOPLoggerPositionAfter]) {
positionOptions=AspectPositionAfter;
}
if ([loggerConfigInfo[AOPLoggerPositionType] isEqualToString:AOPLoggerPositionBefore]) {
positionOptions=AspectPositionBefore;
}

[clazz aspect_hookSelector:selector
withOptions:AspectPositionAfter
usingBlock:^(id<AspectInfo> aspectInfo) {
id log=eventInfo[AOPLoggerLogInfo];

if ([[AOPLogger sharedAOPLogger] conformsToProtocol:objc_getProtocol("AOPLoggerBLLProtocol")]) {
[(AOPLogger<AOPLoggerBLLProtocol>*)[AOPLogger sharedAOPLogger] al_logger:log originAOP:aspectInfo];
}
else{
if ([log isKindOfClass:[NSString class]]) {
NSLog(@"AOPLogger:%@",log);
}
}
} error:NULL];

}
}

}

+(void)AOPLoggerWithClassString:(NSString *)classString methodString:(NSString *)methodString log:(id)log{
[self AOPLoggerWithClassString:classString methodString:methodString log:log logPosition:nil];
}

+(void)AOPLoggerWithClassString:(NSString *)classString methodString:(NSString *)methodString log:(id)log logPosition:(NSString*)logPosition{
Class clazz = NSClassFromString(classString);
SEL selector = NSSelectorFromString(methodString);
AspectOptions positionOptions=AspectPositionAfter;
if ([logPosition isEqualToString:AOPLoggerPositionAfter]) {
positionOptions=AspectPositionAfter;
}
if ([logPosition isEqualToString:AOPLoggerPositionBefore]) {
positionOptions=AspectPositionBefore;
}

[clazz aspect_hookSelector:selector
withOptions:positionOptions
usingBlock:^(id<AspectInfo> aspectInfo) {
if ([[AOPLogger sharedAOPLogger] conformsToProtocol:objc_getProtocol("AOPLoggerBLLProtocol")]) {
[(AOPLogger<AOPLoggerBLLProtocol>*)[AOPLogger sharedAOPLogger] al_logger:log originAOP:aspectInfo];
}
else{
if ([log isKindOfClass:[NSString class]]) {
NSLog(@"AOPLogger:%@",log);
}
}
} error:NULL];

}
```
核心处理.m直接不到百行，主要逻辑就是 单例初始化， 读取plist并格式化出执行的类 ·方法·log信息，最后就是利用AOP,切面hook对应方法插入我们统计逻辑。

有了这个库完全可以日志动态话统计，前提你要做好热更新或使用plist形式。

源码地址：https://github.com/heroims/AOPLogger
