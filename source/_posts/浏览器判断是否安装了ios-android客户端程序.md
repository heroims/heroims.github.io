---
title: 浏览器判断是否安装了ios/android客户端程序
date: 2014-11-04 12:10:31
tags:
    - JavaScript
    - IOS
    - Android
---

<html>

<head>

<meta name="viewport" content="width=device-width" />

</head>

<body>

<h2><a id="applink1" href="mtcmtc://profile/116201417">Open scheme(mtcmtc) defined in iPhone with parameters </a></h2>

<h2><a id="applink2" href="unknown://nowhere">open unknown with fallback to appstore</a></h2>

<p><i>Only works on iPhone!</i></p>



<script type="text/javascript">

// To avoid the "protocol not supported" alert, fail must open another app.

var appstore = "itms://itunes.apple.com/us/app/facebook/id284882215?mt=8&uo=6";

function applink(fail){

return function(){

var clickedAt = +new Date;

// During tests on 3g/3gs this timeout fires immediately if less than 500ms.

setTimeout(function(){

// To avoid failing on return to MobileSafari, ensure freshness!

if (+new Date - clickedAt < 2000){

window.location = fail;

}

}, 500);

};

}

document.getElementById("applink1").onclick = applink(appstore);

document.getElementById("applink2").onclick = applink(appstore);

</script>

</body>

</html>



其原理就是为HTML页面中的超链接点击事件增加一个setTimeout方法.



如果在iPhone上面500ms内，本机有应用程序能解析这个协议并打开程序，则这个回调方法失效；如果本机没有应用程序能解析该协议或者500ms内没有打开个程序，则执行setTimeout里面的function，就是跳转到apple的itunes。


我 用同样的原理来处理android的javascript跳转，发现如果本机没有程序注册intent-filter for 这个协议，那么android内置的browser就会处理这个协议并且立即给出反应(404，你懂的),不会像iPhone一样去执行 setTimeout里面的function，即便你把500ms改成0ms也不管用。

我就开始了我的Google search之旅，最终在stackoverflow一个不起眼的地方找到solution。



不解释，先给出源代码



android里面androidManifest.xml文件对activity的配置，如何配置就不表述了，表达能力有限，请参考developer.android.com



vi<activity android:name=".ui.UploadActivity" android:screenOrientation="portrait">

<intent-filter>

<data android:scheme="http" android:host="192.168.167.33" android:port="8088" android:path="/mi-tracker-web/download.html"/>

<action android:name="android.intent.action.VIEW" />

<category android:name="android.intent.category.DEFAULT" />

<category android:name="android.intent.category.BROWSABLE" />

</intent-filter>

</activity>



HTML页面中指向该应用程序的hyperlink



copy



<a id="applink1" href="http://192.168.167.33:8088/mi-tracker-web/download.html">

Open Application</a>




不难发现，在androidManifest.xml中配置的filter中data的属性表述，在下面的HTML.href中全部看到了。请注意，这两个路径要全部一致，不能有差别，否则android系统就不会拦截这个hyperlink。

好了，为什么我说这个solution能解决我们当初提出来的需求呢，答案在这里：



如果说本机安装了这个应用程序



在android browser中点击HTML中的applink1，browser会重定向到指定的链接，但是由于我们的应用程序在android OS中配置了一个intent-filter，也是针对这个制定的链接。就是说现在android系统有两个程序能处理这个链接：一个是系统的 browser，一个是配置了intent-filter的activity。现在点击这个链接，系统就会弹出一个选择：是用browser还是你指定的 activity打开。如果你选择你的activity，系统就会打开你的应用程序，如果你继续选择用browser，就没有然后了。



如果说本机木有安装这个应用程序



那么这个HTML里面的这个超链接就起很重要的左右了，这个download.html里面可以forward到android的应用商店



download.jsp源代码如下。具体为什么请求的是download.html这个地址却访问到了download.jsp，就不解释了，struts2的东西。



copy



<%@ page language="java" contentType="text/html; charset=ISO-8859-1"

pageEncoding="ISO-8859-1"%>

<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">

<html>

<head>

<meta http-equiv="Content-Type" content="text/html; charset=ISO-8859-1">

<title>Insert title here</title>

</head>

<body>

<script type="text/javascript">

<span style="white-space:pre">    </span>window.location="market://search?q=com.singtel.travelbuddy.android";</script>



view plaincopy

</body>

</html>



在 androidManifest.xml中定义intent-filter的时候定义的scheme，host，port，path拼凑起来是一个有用的 HTTP路径，这样就算本机没有activity定义了intent-filter来捕获这个链接，那这个链接也会重定向到打开android market place的页面，继而打开应用商店。因为每个android手机都会捕获到market这个协议(如果android手机里面没有market商店，不 怪我哈)，系统就会自动打开market place应用商店并根据参数进入搜索页面并显示结果。

