---
title: WebView 常见开发误区
date: 2017-01-01 16:13:04
tags:
    - IOS
    - WebView
    - 常见开发误区
---
# RequestUrl
setRequestUrl方法内注意fragment的概念，当前段采用了#在url中的时候，且作为新页面传给客户端，一定要注意，因为#也就是fragment是不会触发页面重新加载的如果host和path都一样，这时候需要调用reload方法如下，但另一方面也该给前段说明这是启用一个新页面本来就不该用#，背离了用#的意义
```
-(void)setRequestUrl:(NSString *)requestUrl{
_requestUrl=requestUrl;

NSURL *tmpUrl=[NSURL URLWithString:requestUrl];

NSURL *ordUrl=self.lk_webView.request.URL;

[self.lk_webView loadRequest:[NSURLRequest requestWithURL:[NSURL URLWithString:_requestUrl]]];

//fragment不会触发重新加载
if ([tmpUrl.host isEqualToString:ordUrl.host]&&[tmpUrl.path isEqualToString:ordUrl.path]&&![tmpUrl.fragment isEqualToString:ordUrl.fragment]) {
[self.lk_webView reload];
}
}
```
# 异步页面标题设置
通常客户端直接在webview开始加载或加载完js交互调一下document.title设置就完事了，但是如果网页异步设置页面标题就尴尬了，所以通常要定义一个供网页调用的native方法如下，isDynamicTitle控制是否在开始加载和结束加载webview的时候执行`document.title`获取标题动态设置，当前端主动设置就要关闭动态设置
```
-(void)setWebViewTitle:(NSString*)title{
dispatch_async(dispatch_get_main_queue(), ^{
UIViewController *tmpVC=[UIViewController topViewController];
if ([tmpVC isKindOfClass:[LK_WebViewController class]]) {
[(LK_WebViewController*)tmpVC setIsDynamicTitle:NO];
tmpVC.title=title;
}
});
}
```
# 区域内使用webview取消填充
如果在一个固定区域使用webview，你会发现有时底部会有一条线，设背景图背景色都没用，那是view的opaque在生效，需要设置成NO，webview底部会留1像素的缝隙，只有取消填充才能保证背景图背景色不被那条底线干扰
