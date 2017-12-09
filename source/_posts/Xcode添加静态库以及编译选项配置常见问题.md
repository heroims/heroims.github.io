---
title: Xcode添加静态库以及编译选项配置常见问题
date: 2015-06-04 12:17:28
tags:
    - 常见开发误区
    - IOS
    - 静态库
---
# 一,Xcode编译出现Link错误,出现"duplicate symbols for architecture i386 clang"提示.

问题:链接时,项目有重名文件.

解决:

根据错误提示,做如下检查:
1.Taraget->Build Settings->Link Binary With Libraries检查是否有重复lib.

2.全工程搜索下重名文件,决定如何删除.

# 二,关于Category位于静态库时,引用该静态库的工程使用Category,出现"unrecognized selector sent to class"提示.
问题:标准UNIX静态库与Objective-C之间Linker的差异.在标准的UNIX静态库内,linker symbol是依照每一个类别而产生的,但由于Category并没有真正产生一个类别,所以出错.

解决:

1.在该静态库的Taraget->Build Settings->Other Linker Flags->加上 `-ObjC`.

2.在使用该静态库的工程Taraget->Build Settings->Other Linker Flags->加上`-all_load`或`-force_load.`

# 三,编译warning：ld: warning: directory not found for option '-L'.

问题:通常是Path问题.

解决:

Taraget->Build Settings->Library Search Paths 和 Framework Search Paths,删掉编译报warning的路径即OK

# 四,引入(带源码的)静态库所需配置.
步骤:

1.Add Files to.. 加入静态库的.xcodeproj 文件,不要勾选Copy Items.. 选项。(可以先把源代码项目先复制到使用项目文件夹下)

2.Target->Build Phases->Target Dependecies->加静态库 && Link Binary With Libraries->加静态库.

3.配置静态库头文件路径,在Taraget->Build Settings->User Header Search Paths->配上静态库的物理路径.

[错误tips: 若出现加入的.xcodeproj无法展开,则在Xcode中关闭静态库项目即可]

PS:只有.a 和 .h的静态库,则直接拖入项目即可。

