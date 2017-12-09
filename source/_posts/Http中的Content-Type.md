---
title: Http中的Content-Type
date: 2013-02-25 16:16:05
tags:
    - 网络
    - Content-Type
---

HTTP协议（RFC2616）采用了请求/响应模型。客户端向服务器发送一个请求，请求头包含请求的方法、URI、协议版本、以及包含请求修饰符、客户 信息和内容的类似于MIME的消息结构。服务器以一个状态行作为响应，相应的内容包括消息协议的版本，成功或者错误编码加上包含服务器信息、实体元信息以 及可能的实体内容。

通常HTTP消息由一个起始行，一个或者多个头域，一个只是头域结束的空行和可选的消息体组成。HTTP的头域包括通用头，请求头，响应头和实体头四个部分。每个头域由一个域名，冒号（:）和域值三部分组成。域名是大小写无关的，域 值前可以添加任何数量的空格符，头域可以被扩展为多行，在每行开始处，使用至少一个空格或制表符。

请求消息和响应消息都可以包含实体信息，实体信息一般由实体头域和实体组成。实体头域包含关于实体的原信息，实体头包括Allow、Content- Base、Content-Encoding、Content-Language、 Content-Length、Content-Location、Content-MD5、Content-Range、Content-Type、 Etag、Expires、Last-Modified、extension-header。
Content-Type是返回消息中非常重要的内容，表示后面的文档属于什么MIME类型。Content-Type: [type]/[subtype]; parameter。例如最常见的就是text/html，它的意思是说返回的内容是文本类型，这个文本又是HTML格式的。原则上浏览器会根据Content-Type来决定如何显示返回的消息体内容。

type有下面的形式。

Text：用于标准化地表示的文本信息，文本消息可以是多种字符集和或者多种格式的；

Multipart：用于连接消息体的多个部分构成一个消息，这些部分可以是不同类型的数据；

Application：用于传输应用程序数据或者二进制数据；

Message：用于包装一个E-mail消息；

Image：用于传输静态图片数据；

Audio：用于传输音频或者音声数据；

Video：用于传输动态影像数据，可以是与音频编辑在一起的视频数据格式。

subtype用于指定type的详细形式。content-type/subtype配对的集合和与此相关的参数，将随着时间而增长。为了确保这些值在一个有序而且公开的状态下开发，MIME使用Internet Assigned Numbers Authority (IANA)作为中心的注册机制来管理这些值。

parameter可以用来指定附加的信息，更多情况下是用于指定text/plain和text/htm等的文字编码方式的charset参数。MIME根据type制定了默认的subtype，当客户端不能确定消息的subtype的情况下，消息被看作默认的subtype进行处理。Text默认是text/plain，Application默认是application/octet-stream而Multipart默认情况下被看作multipart/mixed。Json常用application/json。

application/x-www-form-urlencoded：数据被编码为名称/值对。这是标准的编码格式。

multipart/form-data： 数据被编码为一条消息，页上的每个控件对应消息中的一个部分

text/plain： 数据以纯文本形式(text/json/xml/html)进行编码，其中不含任何控件或格式字符。postman软件里标的是RAW。


MIME定义在RFC-2046 MIME Part 2: Media Types 。

常用类型：

Mime Types By File Extension

|Extension|Type/sub-type|
|:--:|:--:|
||application/octet-stream|
|323|text/h323|
|acx|application/internet-property-stream|
|ai|application/postscript|
|aif|audio/x-aiff|
|aifc|audio/x-aiff|
|aiff|audio/x-aiff|
|asf|video/x-ms-asf|
|asr|video/x-ms-asf|
|asx|video/x-ms-asf|
|au|audio/basic|
|avi|video/x-msvideo|
|axs|application/olescript|
|bas|text/plain|
|bcpio|application/x-bcpio|
|bin|application/octet-stream|
|bmp|image/bmp|
|c|text/plain|
|cat|application/vnd.ms-pkiseccat|
|cdf|application/x-cdf|
|cer|application/x-x509-ca-cert|
|class|application/octet-stream|
|clp|application/x-msclip|
|cmx|image/x-cmx|
|cod|image/cis-cod|
|cpio|application/x-cpio|
|crd|application/x-mscardfile|
|crl|application/pkix-crl|
|crt|application/x-x509-ca-cert|
|csh|application/x-csh|
|css|text/css|
|dcr|application/x-director|
|der|application/x-x509-ca-cert|
|dir|application/x-director|
|dll|application/x-msdownload|
|dms|application/octet-stream|
|doc|application/msword|
|dot|application/msword|
|dvi|application/x-dvi|
|dxr|application/x-director|
|eps|application/postscript|
|etx|text/x-setext|
|evy|application/envoy|
|exe|application/octet-stream|
|fif|application/fractals|
|flr|x-world/x-vrml|
|gif|image/gif|
|gtar|application/x-gtar|
|gz|application/x-gzip|
|h|text/plain|
|hdf|application/x-hdf|
|hlp|application/winhlp|
|hqx|application/mac-binhex40|
|hta|application/hta|
|htc|text/x-component|
|htm|text/html|
|html|text/html|
|htt|text/webviewhtml|
|ico|image/x-icon|
|ief|image/ief|
|iii|application/x-iphone|
|ins|application/x-internet-signup|
|isp|application/x-internet-signup|
|jfif|image/pipeg|
|jpe|image/jpeg|
|jpeg|image/jpeg|
|jpg|image/jpeg|
|js|application/x-javascript|
|latex|application/x-latex|
|lha|application/octet-stream|
|lsf|video/x-la-asf|
|lsx|video/x-la-asf|
|lzh|application/octet-stream|
|m13|application/x-msmediaview|
|m14|application/x-msmediaview|
|m3u|audio/x-mpegurl|
|man|application/x-troff-man|
|mdb|application/x-msaccess|
|me|application/x-troff-me|
|mht|message/rfc822|
|mhtml|message/rfc822|
|mid|audio/mid|
|mny|application/x-msmoney|
|mov|video/quicktime|
|movie|video/x-sgi-movie|
|mp2|video/mpeg|
|mp3|audio/mpeg|
|mpa|video/mpeg|
|mpe|video/mpeg|
|mpeg|video/mpeg|
|mpg|video/mpeg|
|mpp|application/vnd.ms-project|
|mpv2|video/mpeg|
|ms|application/x-troff-ms|
|mvb|application/x-msmediaview|
|nws|message/rfc822|
|oda|application/oda|
|p10|application/pkcs10|
|p12|application/x-pkcs12|
|p7b|application/x-pkcs7-certificates|
|p7c|application/x-pkcs7-mime|
|p7m|application/x-pkcs7-mime|
|p7r|application/x-pkcs7-certreqresp|
|p7s|application/x-pkcs7-signature|
|pbm|image/x-portable-bitmap|
|pdf|application/pdf|
|pfx|application/x-pkcs12|
|pgm|image/x-portable-graymap|
|pko|application/ynd.ms-pkipko|
|pma|application/x-perfmon|
|pmc|application/x-perfmon|
|pml|application/x-perfmon|
|pmr|application/x-perfmon|
|pmw|application/x-perfmon|
|pnm|image/x-portable-anymap|
|pot|application/vnd.ms-powerpoint|
|ppm|image/x-portable-pixmap|
|pps|application/vnd.ms-powerpoint|
|ppt|application/vnd.ms-powerpoint|
|prf|application/pics-rules|
|ps|application/postscript|
|pub|application/x-mspublisher|
|qt|video/quicktime|
|ra|audio/x-pn-realaudio|
|ram|audio/x-pn-realaudio|
|ras|image/x-cmu-raster|
|rgb|image/x-rgb|
|rmi|audio/mid|
|roff|application/x-troff|
|rtf|application/rtf|
|rtx|text/richtext|
|scd|application/x-msschedule|
|sct|text/scriptlet|
|setpay|application/set-payment-initiation|
|setreg|application/set-registration-initiation|
|sh|application/x-sh|
|shar|application/x-shar|
|sit|application/x-stuffit|
|snd|audio/basic|
|spc|application/x-pkcs7-certificates|
|spl|application/futuresplash|
|src|application/x-wais-source|
|sst|application/vnd.ms-pkicertstore|
|stl|application/vnd.ms-pkistl|
|stm|text/html|
|svg|image/svg+xml|
|sv4cpio|application/x-sv4cpio|
|sv4crc|application/x-sv4crc|
|swf|application/x-shockwave-flash|
|tapplication/x-troff|
|tar|application/x-tar|
|tcl|application/x-tcl|
|tex|application/x-tex|
|texi|application/x-texinfo|
|texinfo|application/x-texinfo|
|tgz|application/x-compressed|
|tif|image/tiff|
|tiff|image/tiff|
|tr|application/x-troff|
|trm|application/x-msterminal|
|tsv|text/tab-separated-values|
|txt|text/plain|
|uls|text/iuls|
|ustar|application/x-ustar|
|vcf|text/x-vcard|
|vrml|x-world/x-vrml|
|wav|audio/x-wav|
|wcm|application/vnd.ms-works|
|wdb|application/vnd.ms-works|
|wks|application/vnd.ms-works|
|wmf|application/x-msmetafile|
|wps|application/vnd.ms-works|
|wri|application/x-mswrite|
|wrl|x-world/x-vrml|
|wrz|x-world/x-vrml|
|xaf|x-world/x-vrml|
|xbm|image/x-xbitmap|
|xla|application/vnd.ms-excel|
|xlc|application/vnd.ms-excel|
|xlm|application/vnd.ms-excel|
|xls|application/vnd.ms-excel|
|xlt|application/vnd.ms-excel|
|xlw|application/vnd.ms-excel|
|xof|x-world/x-vrml|
|xpm|image/x-xpixmap|
|xwd|image/x-xwindowdump|
|z|application/x-compress|
|zip|application/zip|
