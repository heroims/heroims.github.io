---
title: IOS 网络服务层自动化详解
date: 2017-03-02 02:09:05
tags:
    - IOS
    - 网络层
    - 自动化
    - 架构设计
---

# 前言
之前的文章http://www.jianshu.com/p/7a3a387584c6 最后提到了网络服务层自动化，今天在详细说说。之前封装了ServerAPI，这套框架本身其实相当于定义了一套规范的。
自动化的概念还是放最后，先讲清楚这框架，到了后面自动化概念自然而然就出来了。
#ServerAPI 分析与网络服务层构建
框架本身构成主要有4部分
### ServerAPI
主要对后台的API接口做对应描述，相当于网络请求的配置文件Model，定义了请求地址，重试次数，超时时间，返回数据类型，解析数据类型，上传方式等等，，除了常用描述还封装了常用方法算是个标准的胖Model
常用方法例如，请求地址这里也是拆分成host ，path ，pathParameter，parameter，内部封装了组合逻辑，下面4个属性
host：http://xxxx/
path：module/{xxxmudleID}
pathParameter：xxxmudleID:xxxx
parameter：key:value
可以拼成http://xxxx/module/xxxmudleID?key=value 这个url，还有自动为每一次请求生成唯一ID，对请求打Tag方便归类，包括发起请求这也提供了方法，不过实现当然不在这，毕竟他只负责描述


### ServerAPIManager
主要负责管理请求和根据api发起请求，框架内他只是一个管理者逻辑，对发起的请求做归类管理，而发起请求包括缓存逻辑这里通过Category来实现，而具体要实现哪些，由最后面的协议类定义，这样框架本身不去对网络层实现，缓存层实现做干涉，方便更自由的定制化，例如网络层AFN，ASI，还是其他语言都跟框架本身无关，所以这里框架本身只是一个壳，移植性较高，同样的思想拿到其他平台依然适用
### ServerResult
主要把返回的数据用通用型实体表示，包括发起的api，发起请求的request对象，返回的源数据，返回的格式化数据，返回的错误，状态等，由于牵扯到数据格式化，所以这里依然是通过Category实现对应方法，方便扩展

ps：不支持扩展的语言，继承也可以
<!-- more -->
### ServerAPIProtocol
主要负责定义要在Category里实现的方法，这里我直接放源码，注释还算全，关键内容不多，这就是描述要实现哪些才能让框架跑起来，具体方法放那里具体怎么用每个人有每个人不同的想法
```Objective-C
//通过Category对ServerAPI实现对应的协议

@class ServerAPI;

typedef void  (^sap_requestFailHandle)(ServerResult *result, NSError* errInfo);
typedef void  (^sap_requestSuccessHandle)(ServerResult *result);

#pragma mark - 必须实现的协议
@protocol ServerAPIManagerRequestProtocol <NSObject>

@required

/**
发起请求逻辑实现 方便用AFN，ASI或自己写
@param api 请求描述的ServerAPI
@param completion 请求回调
@param progressHandle 请求进度
*/
-(void)requestDataWithAPI:(ServerAPI*)api completion:(sap_requestCompletion)completion progressHandle:(sap_progressHandle)progressHandle;

/**
取消一个请求
@param api 请求描述的ServerAPI
*/
-(void)cancelRequestWithAPI:(ServerAPI*)api;

@optional

/**
发起请求逻辑实现 方便用AFN，ASI或自己写
@param api 请求描述的ServerAPI
@param successHandle 请求成功回调
@param failHandle 请求失败回调
@param progressHandle 请求进度
*/
-(void)requestDataWithAPI:(ServerAPI*)api successHandle:(sap_requestSuccessHandle)successHandle failHandle:(sap_requestFailHandle)failHandle progressHandle:(sap_progressHandle)progressHandle;


/**
根据requestID取消一个请求
@param requestID 请求唯一ID
*/
-(void)cancelRequestWithRequestID:(NSString*)requestID;

/**
根据requestsTag取消某一类表示的请求
@param requestsTag 请求分类标识
*/
-(void)cancelRequestWithRequestsTag:(NSString*)requestsTag;

/**
根据requestID数组取消相关请求
@param requestList requestID的数组
*/
-(void)cancelRequestWithRequestIDList:(NSArray*)requestList;

/**
根据ServerAPI数组取消相关请求
@param requestList ServerAPI的数组
*/
-(void)cancelRequestWithAPIList:(NSArray*)requestList;

/**
取消最后一个请求
*/
-(void)cancelLastRequest;

/**
取消第一个请求
*/
-(void)cancelFirstRequest;

@end

@protocol ServerAPIResponseProtocol <NSObject>

@required

/**
请求结束时间   最好设置时放到responseFormatWithData方法内
@return 请求结束时间
*/
-(NSDate *)endDate;

/**
数据格式化处理
@param data 数据源
@param error 错误源
@param completion 请求回调
@param cacheData 缓存数据
*/
-(void)responseFormatWithData:(id)data error:(NSError*)error completion:(sap_requestCompletion)completion cacheData:(id)cacheData;
@optional

/**
数据格式化后的处理 用于对ServerAPI实现Category加入部分业务逻辑
@param result 格式化数据
*/
-(void)responseCustomDoInCategoryWithResult:(ServerResult*)result;

@end

#pragma mark - 可选协议
@protocol ServerAPIManagerCacheProtocol <NSObject>

@optional

/**
拉取缓存数据
@param api 请求描述的ServerAPI
@param completion 请求回调
@param error 错误信息
@return 缓存有无
*/
-(BOOL)fetchDataCacheWithAPI:(ServerAPI*)api completion:(sap_requestCompletion)completion error:(NSError*)error;

/**
拉取缓存数据
@param api 请求描述的ServerAPI
@param successHandle 请求成功回调
@param failHandle 请求失败回调
@param error 错误信息
@return 缓存有无
*/
-(BOOL)fetchDataCacheWithAPI:(ServerAPI*)api successHandle:(sap_requestCompletion)successHandle failHandle:(sap_requestFailHandle)failHandle error:(NSError*)error;

/**
保存缓存
@param api 请求描述的ServerAPI
*/
-(void)saveDataCacheWithResult:(ServerAPI*)api;
@end

/**
与ServerAPIRequstProtocol的协议有重叠主要用于内部封装逻辑
*/
@protocol ServerAPIRequstOptionalProtocol <NSObject>

@optional
-(void)requestDataWithSuccessHandle:(sap_requestCompletion)successHandle failHandle:(sap_requestFailHandle)failHandle;

+(ServerAPI*)newRequestDataWithSuccessHandle:(sap_requestSuccessHandle)successHandle failHandle:(sap_requestFailHandle)failHandle;

@end

/**
与ServerAPIManagerRequestProtocol的协议有重叠主要用于内部封装逻辑
*/
@protocol ServerAPIManagerRequestOptionalProtocol <NSObject>

@optional
-(void)requestDataWithAPI:(ServerAPI*)api completion:(sap_requestCompletion)completion progressHandle:(sap_progressHandle)progressHandle;

-(void)requestDataWithAPI:(ServerAPI*)api successHandle:(sap_requestSuccessHandle)successHandle failHandle:(sap_requestFailHandle)failHandle;

-(void)cancelRequestWithAPI:(ServerAPI*)api;
-(void)cancelRequestWithRequestID:(NSString*)requestID;
-(void)cancelRequestWithRequestsTag:(NSString*)requestsTag;
-(void)cancelRequestWithRequestIDList:(NSArray*)requestList;
-(void)cancelRequestWithAPIList:(NSArray*)requestList;
-(void)cancelLastRequest;
-(void)cancelFirstRequest;

@end
```
源码地址：https://github.com/heroims/ServerAPI/
#ServerAPI使用
下面说说怎么用，按这种模式填充完自己定制的方法实现后，既可以当做离散型API用也会可以当做集约型API用，还可以直接用URL
```Objective-C
//离散型API使用  一个请求是一个API类，用NSClassFromString主要为了省头文件引用懒得加头文件。。。           
[NSClassFromString(@"DemoAPI") newRequestDataWithCompletion:^(ServerResult *result, NSError *errInfo) {

} requestParameters:nil requestTag:NSStringFromClass([self class])];

ServerAPI *discreteApi=[[NSClassFromString(@"DemoAPI") alloc] init];
discreteApi.requestParameters=nil;
discreteApi.requestTag=NSStringFromClass([self class]);
[discreteApi requestDataWithCompletion:^(ServerResult *result, NSError *errInfo) {

}];

//集约型API使用  可以直接类扩展加静态方法设置，可以自行类扩展个更简洁通用的方法这里只距离
ServerAPI *intensiveAPI=[[ServerAPI alloc] init];
intensiveAPI.requestHost=@"http://xxx.xxx.xxx";
intensiveAPI.requestPath=@"xxx";
intensiveAPI.requestParameters=nil;
intensiveAPI.requestTag=NSStringFromClass([self class]);
intensiveAPI.accessType=APIAccessType_Get;
intensiveAPI.resultFormat=APIResultFormat_JSON;
intensiveAPI.timeOut=30;
intensiveAPI.retryTimes=2;
[intensiveAPI requestDataWithCompletion:^(ServerResult *result, NSError *errInfo) {

}];

//直接请求Url
ServerAPI *urlAPI=[[ServerAPI alloc] init];
urlAPI.requestURL=@"http://xxx.xxx.xxx";
urlAPI.requestTag=NSStringFromClass([self class]);
[urlAPI requestDataWithCompletion:^(ServerResult *result, NSError *errInfo) {

}];

```
# ServerAPI自动化产出（离散型API好处）
当采用了离散型API时，调用的时候通常只需要初始化对应的API，传入后台要的参数，至此数据返回，一个请求完成。
通过继承ServerAPI，return 描述的固定值一个对应的API类就创建完成，然后回到之前的文章，如果服务端写的代码工整或有地方统一定义直接可以自动导出一套API的所有描述文件，也就是各各XXXAPI，哪怕代码不工整，API文档总该有吧，有点格式就可以读出来自动生成API，如果还没有，，，，好吧，那就帮他们规范化所以有了下面我用Swift写的工具 。
#### ServerAPICreator
https://github.com/heroims/ServerAPICreator
功能很简单，就是对API接口的描述，填完自动生成wiki.md和一堆XXXAPI.h,XXXAPI.m文件。
这种东西就是定制化比较高的东西了，所以代码也就看看吧，拿来直接用是不可能的，怎样都要改改，但思路相信看到这的人基本都有了

![687474703a2f2f6865726f696d732e6769746875622e696f2f53657276657241504943726561746f722f32414530344336422d364233422d343738332d393537322d4143423746333632314641432e706e67.png](/assets/blogImage/3994053-254e821d28eb7c80.png)

### 联想
按照这种设计架构的方式很多层面的东西都会渐渐的用描述解决，开发一套东西需要的是脚本描述。核心引擎好了，想想人工智能是不是不断积累数据，产出脚本描述然后不断的自动写代码给自己打patch（随便一说扯扯淡）。。。。
