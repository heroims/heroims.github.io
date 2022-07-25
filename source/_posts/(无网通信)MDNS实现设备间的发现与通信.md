---
title: (无网通信)MDNS实现设备间的发现与通信
date: 2021-12-01 00:09:05
tags:
    - Wifi Direct
    - NDS
    - MDNS
    - DNS-SD
    - iOS
    - Andoid
    - 无网通信
    - BLE
typora-root-url: ../
---

# 简介
这里无网通信指的是没有互联网的通信方式，之前在区块链公司是做这块，现在怕自己以后忘了写一遍，之前搞工业机器人实在太忙，整个博客停了，也不搞移动了，现在离职了，创业也败了，该恢复一下之前的技术调整心态继续找工作了。

所谓无网通信其实是不同设备互相发现然后连接进行通信，而互相发现的方式可以归纳成三种

- 两个设备连接在同一个WIFI
- 两个设备通过WLAN创建点对点连接
- 两个设备通过蓝牙BLE创建点对点连接

而前两种是最稳定的方式，可以使用Wifi Direct实现，而蓝牙BLE的方式就比较鸡肋了距离有限制大小也有限制，能Wifi直连肯定不用蓝牙。然而蓝牙的发现连接标准其实各个设备基本都遵循的是一套，但Wifi Direct则是实现方式多种多样像UPnP、Bonjour、DLNA、SLP抑或是其他技术都能实现。iOS用的自家Bonjour，而Android4.0后开始支持直接就叫Wifi Direct，不同设备直连基本都会遇到问题。这里以Android和iOS为例实现的基本逻辑其实都由MDNS与DNS-SD演化而来。

确定了Wifi Direct以后又会发现一个问题就是这解决的是设备与设备的通信，也就是一对一，而理想的无网通信是多对多，信息流之间希望有多跳的能力也就是Mesh，这个最终我们也没完美解决，虽然Android和iOS里有一定的解决方案(创建群组实现多对多)但也有不同系统无法互通的问题和群组上限问题，但当时我们选择了更简单的方式就是由路由器去Mesh其他路由器，而设备加入到路由器的网络里直接用MDNS去发现然后链接通信，专业设备做专业事，Mesh组网和设备通信分摊到两个地方。

## MDNS(Multicast DNS)与DNS-SD(DNS Service Discovery)

简介里介绍完基本就确定了技术点就是Wifi Direct，而这项技术则是由MDNS(Multicast DNS)与DNS-SD(DNS Service Discovery)演化而来。

首先MDNS和DNS-SD是DNS协议的两个扩展。MDNS扩展了域名服务系统，在链路本地多播上运行。DNS-SD添加了通过DNS发现网络服务的支持。MDNS原理就是在基于udp加入网络后向所有主机组播一个消息。

基本流程就是基于某个自定义域名发布服务，发现设备，连接设备

![Bonjour API](/assets/blogImage/WX20211206-012426@2x.png "Bonjour API")

<!-- more -->

## Andoid

安卓官网写的很详细非常友好

#### [独立设备下Andoid NSD](https://developer.android.com/training/connect-devices-wirelessly/nsd-wifi-direct)

这里讲的是不加入到任何共同网络中的独立设备，通过android的Wifi Direct实现启动服务发现设备而且是有条件的发现，类似MDNS可以指定启动及发现服务的名称

#### [在同一网络下Andoid NSD](https://developer.android.com/training/connect-devices-wirelessly/nsd)

这里讲的是在设备加入到同一网络中发现设备，而这块则是最标准最通用的，遵循MDNS协议实现设备的发现解析出host，我们当时就是运用这个使得Android，iOS，其他设备都能在网络下互相发现我们的服务，其他设备用Golang的MDNS，github能搜到很多封装好的，而且区块链著名的二层网络Rainden Network的节点发现也有用到它。像智能设备的控制基本都可以选择这个方案解决

#### [Andoid WIFI直连](https://developer.android.com/training/connect-devices-wirelessly/wifi-direct)

这里主要讲的是第一个类似，但他是无差别的去发现附近开启对等连接的设备，然后还讲了分组和连接有点对标iOS MutipeerConnectivity的意思。



## iOS

首先苹果官网想通过关键词搜索相关技术你没看过WWDC基本上搜不出来，尤其是近几年的东西你就算搜出来，越是抽象的API越是没啥注释，越是看注释就知道怎么用的，它还给你来个示例代码。下面开始详细说说:

iOS主要提供了3个方式MutipeerConnectivity，[Network库](https://developer.apple.com/documentation/network?language=objc)，[NSNetService相关](https://developer.apple.com/search/?q=NSNetService),而我当时用的是NSNetService因为那个时候还没有Network库，现在这些类都已经被淘汰了，而MutipeerConnectivity虽然方法封装的更好用而且同时支持蓝牙和wifi直连，但你没法控制代码去控制用哪个，而且由于太标准无法做和android的通信，但如果是仅仅iOS间通信聊天，却能直接轻松实现群组间的聊天。

#### [MutipeerConnectivity](https://developer.apple.com/library/archive/navigation/#section=Technologies&topic=MultipeerConnectivity)

这示例代码实现了无网聊天在不加入网络的情况下发现附近的群组加入进行聊天，但是直接看代码还是有点费劲不向上面Android的文档讲了关键的几个方法，所以我就依照示例和常用的挑出核心代码简单讲一下。

```objective-c
		//创建一个设备ID
		MCPeerID *peerID;
    @try {
        peerID = [[MCPeerID alloc] initWithDisplayName:self.displayNameTextField.text];
    }
    @catch (NSException *exception) {
        NSLog(@"Invalid display name [%@]", self.displayNameTextField.text);
        return NO;
    }
```



```objective-c
		@try {
      	//创建广播对象，serviceType相当于房间名称，discoveryInfo是广播信息，这个方法相当于告诉附近的您愿意加入指定类型的回话,在委托里可以处理各种自定义逻辑
        advertiser = [[MCNearbyServiceAdvertiser alloc] initWithPeer:peerID discoveryInfo:nil serviceType:self.serviceTypeTextField.text];
    }
    @catch (NSException *exception) {
        NSLog(@"Invalid service type [%@]", self.serviceTypeTextField.text);
        return NO;
    }
```



```objective-c
#pragma mark - MCNearbyServiceAdvertiserDelegate
//可以通过invitationHandler传入accept进行连接会话结果过滤.
- (void)            advertiser:(MCNearbyServiceAdvertiser *)advertiser
  didReceiveInvitationFromPeer:(MCPeerID *)peerID
                   withContext:(nullable NSData *)context
             invitationHandler:(void (^)(BOOL accept, MCSession * __nullable session))invitationHandler;

```





```objective-c
  // 开启广播
	[advertiser startAdvertisingPeer];
	// 停止广播
  [advertiser stopAdvertisingPeer];
```



```objective-c
        //创建回话对象，用于存放当前连接的会话
        _session = [[MCSession alloc] initWithPeer:peerID securityIdentity:nil encryptionPreference:MCEncryptionRequired];
        // 设置 MCSessionDelegate 
        _session.delegate = self;
        // 创建广播对象，和之前的不同在于提供了接收邀请的标准界面
        _advertiserAssistant = [[MCAdvertiserAssistant alloc] initWithServiceType:serviceType discoveryInfo:nil session:_session];
        // 开启广播
        [_advertiserAssistant start];
```



```objective-c
		//停止广播
    [_advertiserAssistant stop];
  	//断开会话连接
    [_session disconnect];
```



```objective-c
  //发送消息给当前会话中的对象  
	[self.session sendData:messageData toPeers:self.session.connectedPeers withMode:MCSessionSendDataReliable error:&error];
```



```objective-c
		NSProgress *progress;
    // 遍历连接的设备ID
    for (MCPeerID *peerID in _session.connectedPeers) {
				//        imageUrl = [NSURL URLWithString:@"http://images.apple.com/home/images/promo_logic_pro.jpg"];
        // 发送图片给对应设备
        progress = [self.session sendResourceAtURL:imageUrl withName:[imageUrl lastPathComponent] toPeer:peerID withCompletionHandler:^(NSError *error) {
            // Implement this block to know when the sending resource transfer completes and if there is an error.
            if (error) {
                NSLog(@"Send resource to peer [%@] completed with Error [%@]", peerID.displayName, error);
            }
            else {
                // Create an image transcript for this received image resource
                Transcript *transcript = [[Transcript alloc] initWithPeerID:_session.myPeerID imageUrl:imageUrl direction:TRANSCRIPT_DIRECTION_SEND];
                [self.delegate updateTranscript:transcript];
            }
        }];
    }
```



```objective-c
#pragma mark - MCSessionDelegate
//会话接受数据的时候调用
- (void)session:(MCSession *)session didReceiveData:(NSData *)data fromPeer:(MCPeerID *)peerID
{
    
}
//会话状态发生改变的时候调用
- (void)session:(MCSession *)session peer:(MCPeerID *)peerID didChangeState:(MCSessionState)state
{
  
}
```



```objective-c
  	//创建搜索蓝牙设备控制器  
		MCBrowserViewController *browserViewController = [[MCBrowserViewController alloc] initWithServiceType:self.serviceType session:self.sessionContainer.session];
                                                      
		browserViewController.delegate = self;
    browserViewController.minimumNumberOfPeers = kMCSessionMinimumNumberOfPeers;
    browserViewController.maximumNumberOfPeers = kMCSessionMaximumNumberOfPeers;

    [self presentViewController:browserViewController animated:YES completion:nil];

```



```objective-c
#pragma mark - MCBrowserViewControllerDelegate
//搜索到设备后进行筛选用的回调
- (BOOL)browserViewController:(MCBrowserViewController *)browserViewController shouldPresentNearbyPeer:(MCPeerID *)peerID withDiscoveryInfo:(NSDictionary *)info
{
    return YES;
}

//搜索界面点击完成的时候调用
- (void)browserViewControllerDidFinish:(MCBrowserViewController *)browserViewController
{
    [browserViewController dismissViewControllerAnimated:YES completion:nil];
}

//搜索界面点击取消的时候调用
- (void)browserViewControllerWasCancelled:(MCBrowserViewController *)browserViewController
{
    [browserViewController dismissViewControllerAnimated:YES completion:nil];
}
```



上面就是实现聊天功能的一些代码片段，但反过来想一下，这个功能主要是实现发现附近设备加入聊天室聊天，这场景其实比较奇怪，离得不远的话聊两句破冰了肯定流传了毕竟离得不远直接面对面不是更好，只有在游戏场景里这种近点通信才是一个比较好的交流方式，比如游戏道具的互换，游戏竞技比赛这类场景跟这类技术才会比较搭配，聊天的话真的就当个例子就好。。。

#### [NSNetService](https://developer.apple.com/library/archive/samplecode/WiTap)

这里示例代码实现的是使用NSNetService实现两个设备连接后点击色块反馈，现在这个类已经被弃用代替方法是Network库。

这里简单简绍就不已经示例代码了，因为这块可以直接实现和安卓的互通所以就以我个人总结的几个关键地方为主。

安卓方面使用就是 [在同一网络下Andoid NSD](https://developer.android.com/training/connect-devices-wirelessly/nsd) 叙述的设置

```objective-c
//初始化广播服务为了和Android通信Domain为空字符串，其他的Android上都有，至于port就是你起服务的Socket端口
_netServiceServer = [[NSNetService alloc] initWithDomain:self.netServiceDomain
                                                            type:self.netServiceType
                                                            name:self.netServerName
                                                            port:self.tcpSocketServer.localPort];
```



```objective-c
#pragma mark - Bonjour

//发布服务
- (BOOL) publishService {
    [self.netServiceServer scheduleInRunLoop:[NSRunLoop currentRunLoop]
                                     forMode:NSRunLoopCommonModes];
    [self.netServiceServer setDelegate:self];
    [self.netServiceServer publish];
    
    return YES;
}

//停止服务
- (void) unpublishService {
    if (self.netServiceServer) {
        [self.netServiceServer stop];
        [self.netServiceServer removeFromRunLoop:[NSRunLoop currentRunLoop] forMode:NSRunLoopCommonModes];
        self.netServiceServer = nil;
    }
}
```



```objective-c
    //设置委托
		netService.delegate=self;
		//解析地址 超时时间为5秒
    [netService resolveWithTimeout:5];
```



```objective-c
#pragma mark - NSNetServiceDelegate
//发布服务失败的时候调用
-(void)netService:(NSNetService *)sender didNotPublish:(NSDictionary<NSString *,NSNumber *> *)errorDict{
    
}

//地址解析完成的时候调用
-(void)netServiceDidResolveAddress:(NSNetService *)sender{

}
```



```objective-c
//启动发现服务，之前测试停掉再启动就不生效了，所以每次都重新初始化
- (BOOL)startWithServicesOfType:(NSString*)type inDomain:(NSString*)domain{
    if ( netServiceBrowser != nil ) {
        [self stop];
    }
    
    netServiceBrowser = [[NSNetServiceBrowser alloc] init];
    if( !netServiceBrowser ) {
        return NO;
    }
    
    netServiceBrowser.delegate = self;
  	//设置搜索的参数，与Android通信设置domain为空字符串
    [netServiceBrowser searchForServicesOfType:type inDomain:domain];
    
    return YES;
}

//停止发现服务
- (void)stop {
    if ( netServiceBrowser == nil ) {
        return;
    }
    
    [netServiceBrowser stop];
    netServiceBrowser = nil;
    
    [_servers removeAllObjects];
}
```

通过上面代码就能实现iOS和Android发现彼此设备的IP和端口，只要你起好tcp的server就能进行连接通信，至于起server随便百度一下就有很多socket起服务的方法这里就不做赘述了我是直接用[CocoaAsyncSocket](https://github.com/robbiehanson/CocoaAsyncSocket)，我当时为了保险做成了混合式发现入网后还会定期udp广播一下其实没必要。

#### [Network库](https://developer.apple.com/documentation/network?language=objc)

这个没在官方找到什么好的示例代码所以直接就给出了官方文档，在网上看到过不少散装的，NSNetService对应NWConnection初始化用NWEndpoint来初始化，NSNetServiceBrowser对应NWBrowser

```swift
import Network

let parameter = NWParameters()
//之前的属性NSNetServiceBrowser NSNetService现在都用NWParameters这个对象来传递
parameter.includePeerToPeer = true
let browser = NWBrowser(for: .bonjour(type: "services.dns-sd.", domain: nil), using: parameter)
//状态改变的时候回调
browser.stateUpdateHandler = { state in
    switch state {
    case .ready:
        print("ready")
    case .failed(let error):
        print("error:", error.localizedDescription)
    default:
        break
    }
}
//结果有变化的时候回调
browser.browseResultsChangedHandler = { result, changed in
    result.forEach { device in
        //endpoint中有netServiceName netServiceType 
        print(device.endpoint)
        print(device.metadata)
    }
}
//主线程上启动搜索服务
browser.start(queue: .main)
```



```swift
      //初始化tcp server，入参对应NSNetService初始化的
			self.listener = try NWListener(using: .tcp, on :localPort)
      self.listener?.service = NWListener.Service(name:netServiceName, type: netServiceType, domain: serviceDomain, txtRecord: nil)
      //注册服务后回调，对应NSNetService发布服务成功后的回调
      self.listener?.serviceRegistrationUpdateHandler = { (serviceChange) in
        switch(serviceChange) {
        case .add(let endpoint)://设备加入
          switch endpoint {
          case let .service(name, type, domain, interface):
            print("Service Name \(name) of type \(type) having domain: \(domain) and interface: \(String(describing: interface?.debugDescription))")
          default:
            break
          }
        default:
          break
        }
      }
      self.listener?.stateUpdateHandler = {(newState) in
        switch newState {
        case .ready:
            print("Bonjour TCP Listener: Bonjour listener state changed - ready")
        default:
          break
        }
      }
      self.listener?.newConnectionHandler = {(newConnection) in
        newConnection.stateUpdateHandler = {newState in
          switch newState {
          case .ready:
            print("Bonjour TCP Listener: new  connection state - ready")

            
          default:
            break
          }
        }
        newConnection.start(queue: DispatchQueue(label: "Bonjour TCP Listener: New Connection"))
      }
    } catch {
      print("Bonjour TCP Listener: Unable to create listener")
    }
    self.listener?.start(queue: .main)

```



```swift
	//初始化tcp client
	self.netConnect = NWConnection(to: .service(name: netServiceName, type: netServiceType, domain: serviceDomain, interface: nil), port:localPort, using: .tcp)
	self.netConnect?.stateUpdateHandler = { (newState) in
       print("bonjourToTCP: Connection details: \(String(describing: self.netConnect?.debugDescription))")
     switch (newState) {
     case .ready:
        self.connectState = "Connection state: Ready"
       print("bonjourToTCP: new TCP connection ready ")
     default:
       break
     }
	}
	self.netConnect?.start(queue: .main)
```

新的Network库好处是直接把socket和注册广播服务整合了，这个也没真正测过在这里只是做个记录。

## 其他平台

### Golang

#### [mDNS](https://github.com/hashicorp/mdns)

#### [bonjour](https://github.com/oleksandr/bonjour)

### Node.js

#### [mDNS](https://github.com/agnat/node_mdns)

### C

#### [mDNS](https://github.com/mjansson/mdns)





