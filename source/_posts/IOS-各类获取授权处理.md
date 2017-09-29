---
title: IOS 各类获取授权常用处理
date: 2017-09-29 16:50:41
tags:
    - 实用技巧
    - IOS
    - 授权
    - 地理位置
    - 网络判断
    - 相册
    - 相机
    - 麦克风
    - 通讯录
    - 蜂窝网络
---

# 地理位置授权
需要引用`#import <CoreLocation/CoreLocation.h>`
- plist
```
始终允许访问位置
NSLocationAlwaysUsageDescription
位置
NSLocationUsageDescription
在使用期间访问位置
NSLocationWhenInUseUsageDescription
始终允许访问位置并且在使用期间访问位置
NSLocationAlwaysAndWhenInUseUsageDescription
```
- 判断
```
//判断是否开启
[CLLocationManager locationServicesEnabled];
//获取具体状态
[CLLocationManager authorizationStatus];
```
- 授权
```
CLLocationManager *locManager = [[CLLocationManager alloc] init];
[locManager ]
//获取始终允许访问位置权限
[locManager requestWhenInUseAuthorization];
//获取在使用期间访问位置
[locManager requestAlwaysAuthorization];
```
# 相册授权
需要引用`#import <AssetsLibrary/AssetsLibrary.h>`或`#import <Photos/Photos.h>`
- plist
```
访问相册
NSPhotoLibraryUsageDescription
添加相册
NSPhotoAddLibraryUsageDescription
```
- 判断
```
[ALAssetsLibrary authorizationStatus];

[PHPhotoLibrary authorizationStatus];
```
- 授权
```
[PHPhotoLibrary requestAuthorization:^(PHAuthorizationStatus status) {
if (status == PHAuthorizationStatusAuthorized) {
// 用户同意授权
}
else {
// 用户拒绝授权
}
}];
```
# 相机&麦克风授权
需要引用`#import <AVFoundation/AVFoundation.h>`
- plist
```
相机
NSCameraUsageDescription
麦克风
NSMicrophoneUsageDescription
```
- 判断
```
//AVMediaTypeVideo相机 AVMediaTypeAudio麦克风
[AVCaptureDevice authorizationStatusForMediaType:AVMediaTypeVideo];
```
- 授权
```
//AVMediaTypeVideo相机 AVMediaTypeAudio麦克风
[AVCaptureDevice requestAccessForMediaType:AVMediaTypeVideo completionHandler:^(BOOL granted) {

if (granted){// 用户同意授权

}
else {// 用户拒绝授权

}
}];

//不推荐慎用
[[AVAudioSession sharedInstance] performSelector:@selector(requestRecordPermission:) withObject:^(BOOL granted) {
if (granted){// 用户同意授权

}
else {// 用户拒绝授权

}
}];
```
# 通讯录授权
需要引用`#import <Contacts/Contacts.h>`
- plist
```
通讯录
NSContactsUsageDescription
```
- 判断
```
[CNContactStore authorizationStatusForEntityType:CNEntityTypeContacts]
```
- 授权
```
CNContactStore*contactStore = [[CNContactStore alloc] init];

[contactStore requestAccessForEntityType:CNEntityTypeContacts completionHandler:^(BOOL granted,NSError*_Nullable error) {
if (granted){// 用户同意授权

}
else {// 用户拒绝授权

}
}];
```
<!-- more -->
# 蜂窝网络授权
### 目前无解，可能以后也无解

仅可以判断首次安装是不是蜂窝网络没开或没授权

引用以下头文件
```
#import <SystemConfiguration/CaptiveNetwork.h>
#import <SystemConfiguration/SCNetworkReachability.h>
#import <CoreTelephony/CTTelephonyNetworkInfo.h>
#import <CoreTelephony/CTCellularData.h>
#import <netinet/in.h>
```
- 判断
```
//info不为nil，则当前链接wifi
- (NSDictionary *)fetchSSIDInfo {
NSArray *ifs = (__bridge_transfer NSArray *)CNCopySupportedInterfaces();
if (!ifs) {
return nil;
}

NSDictionary *info = nil;
for (NSString *ifnam in ifs) {
info = (__bridge_transfer NSDictionary *)CNCopyCurrentNetworkInfo((__bridge CFStringRef)ifnam);
if (info && [info count]) { break; }
}
return info;
}

//返回CTRadioAccessTechnologyGPRS时说明是2g，网络情况复杂就当无网
- (NSString *)fetchMobileInfo {
CTTelephonyNetworkInfo *info = [[CTTelephonyNetworkInfo alloc] init];
return  info.currentRadioAccessTechnology;
}

- (BOOL)checkNetworkConnect {
// 创建零地址，0.0.0.0的地址表示查询本机的网络连接状态
struct sockaddr_in zeroAddress;//sockaddr_in是与sockaddr等价的数据结构
bzero(&zeroAddress, sizeof(zeroAddress));
zeroAddress.sin_len = sizeof(zeroAddress);
zeroAddress.sin_family = AF_INET;//sin_family是地址家族，一般都是“AF_xxx”的形式。通常大多用的是都是AF_INET,代表TCP/IP协议族

/**
*  SCNetworkReachabilityRef: 用来保存创建测试连接返回的引用
*
*  SCNetworkReachabilityCreateWithAddress: 根据传入的地址测试连接.
*  第一个参数可以为NULL或kCFAllocatorDefault
*  第二个参数为需要测试连接的IP地址,当为0.0.0.0时则可以查询本机的网络连接状态.
*  同时返回一个引用必须在用完后释放.
*  PS: SCNetworkReachabilityCreateWithName: 这个是根据传入的网址测试连接,
*  第二个参数比如为"www.apple.com",其他和上一个一样.
*
*  SCNetworkReachabilityGetFlags: 这个函数用来获得测试连接的状态,
*  第一个参数为之前建立的测试连接的引用,
*  第二个参数用来保存获得的状态,
*  如果能获得状态则返回TRUE，否则返回FALSE
*
*/
SCNetworkReachabilityRef defaultRouteReachability = SCNetworkReachabilityCreateWithAddress(NULL, (struct sockaddr *)&zeroAddress); //创建测试连接的引用：
SCNetworkReachabilityFlags flags;

BOOL didRetrieveFlags = SCNetworkReachabilityGetFlags(defaultRouteReachability, &flags);
CFRelease(defaultRouteReachability);
if (didRetrieveFlags && flags == 0) {
//当前是没有打开网络情况进入,可能是关闭蜂窝/无线或进入飞行模式
return NO;
}
return YES;
}

- (void)startValidateNetworkAuthorization:(void(^)(CTCellularDataRestrictedState state))block {
CTCellularData *cellularData = [[CTCellularData alloc]init];
cellularData.cellularDataRestrictionDidUpdateNotifier =  ^(CTCellularDataRestrictedState state){
block(state);
//获取联网状态
switch (state) {
case kCTCellularDataRestricted:

break;
case kCTCellularDataNotRestricted:

break;
case kCTCellularDataRestrictedStateUnknown:

break;
default:
break;
};
};
}
```
- 调用
```
BOOL checkConnect=YES;
BOOL isConnectWifi=[self fetchSSIDInfo];
BOOL isGPRS=[[self fetchMobileInfo] isEqualToString:CTRadioAccessTechnologyGPRS];
BOOL isConnectNetwork=[self checkNetworkConnect];
if (!isConnectWifi) {
checkConnect=NO;
}
if (isGPRS) {
checkConnect=NO;
}
if (!isConnectNetwork) {
checkConnect=NO;
}

if (![[[NSUserDefaults standardUserDefaults] objectForKey:@"firstInstall"] boolValue]) {
[[NSUserDefaults standardUserDefaults] setObject:@(YES) forKey:@"firstInstall"];
if (checkConnect) {
[self startValidateNetworkAuthorization:^(CTCellularDataRestrictedState state) {
if (state != kCTCellularDataRestricted) {
//首次打开app，未连接wifi移动网络正常，即有网情况蜂窝关闭，需提示用户开启蜂窝网络
}
}];
}
}
```
# 其他授权
- plist
```
蓝牙
NSBluetoothPeripheralUsageDescription
媒体资料库
NSAppleMusicUsageDescription
音乐
kTCCServiceMediaLibrary
Siri
NSSiriUsageDescription
语音识别
NSSpeechRecognitionUsageDescription
日历
NSCalendarsUsageDescription
健康分享
NSHealthShareUsageDescription
健康更新
NSHealthUpdateUsageDescription
运动与健康
NSMotionUsageDescription
提醒事项
NSRemindersUsageDescription
FaceID
NSFaceIDUsageDescription
智能家居
NSHomeKitUsageDescription
运动
NSMotionUsageDescription
NFC
NFCReaderUsageDescription
视频认证
NSVideoSubscriberAccountUsageDescription
```
