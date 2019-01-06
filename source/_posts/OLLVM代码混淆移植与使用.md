---
title: OLLVM代码混淆移植与使用
date: 2019-01-06 02:09:05
tags:
    - LLVM
    - OLLVM
    - Clang
    - 混淆
    - XCode
    - NDK
    - Visual Studio
---
# 简介
OLLVM(Obfuscator-LLVM)是瑞士西北应用科技大学安全实验室于2010年6月份发起的一个项目,该项目旨在提供一套开源的针对LLVM的代码混淆工具,以增加对逆向工程的难度。github上地址是https://github.com/obfuscator-llvm/obfuscator，只不过仅更新到llvm的4.0，2017年开始就没在更新。
# 移植
OLLVM如果自己想拿最新版的LLVM和Clang进行移植功能其实也并不是很难，整理一下其实改动很小，接下来将会讲一下移植的方法。

## 个人整理
先放一下个人移植好的版本地址https://github.com/heroims/obfuscator.git，个人fork原版后又加入了llvm5.0，6.0，7.0以及swift-llvm5.0的版本，应该能满足大部分需求了，如果有新版本下面的讲解，各位也可以自己动手去下载自己需要的llvm和clang进行移植。git上的提交每次都很独立如下图，方便各位cherry-pick。

![image.png](/assets/blogImage/3994053-1d4286b24563a1db.png)


## 下载LLVM
llvm地址：https://github.com/llvm-mirror
swift-llvm地址：https://github.com/apple
大家可以从上面的地址下载最新的自己需要的llvm和clang

``` shell
#下载llvm源码
wget https://codeload.github.com/llvm-mirror/llvm/zip/release_70
unzip llvm-release_70.zip
mv llvm-release_70 llvm


#下载clang源码
wget https://codeload.github.com/llvm-mirror/clang/zip/release_70
unzip clang-release_70.zip
mv clang-release_70 llvm/tools/clang

```
##  添加混淆代码
如果用git的话只需要执行`git cherry-pick xxxx`把xxxx换成对应的我的版本上的提交哈希填上即可。极度推荐用git搞定。

如果手动一点点加的话，第一步就是把我改过的OLLVM文件夹里`/include/llvm/Transforms/Obfuscation`和`/lib/llvm/Transforms/Obfuscation`移动到刚才下载好的llvm源码文件夹相同的位置。
``` shell
git clone https://github.com/heroims/obfuscator.git
cd obfuscator
git checkout llvm-7.0
cp include/llvm/Transforms/Obfuscation llvm/include/llvm/Transforms/Obfuscation
cp lib/llvm/Transforms/Obfuscation llvm/lib/llvm/Transforms/Obfuscation
```
然后手动修改8个文件如下：

![image.png](/assets/blogImage/3994053-3d3054a05c96b72a.png)

![image.png](/assets/blogImage/3994053-e76e7039112a47c8.png)

![image.png](/assets/blogImage/3994053-d3c16047c368349d.png)

![image.png](/assets/blogImage/3994053-79683114fe1a73c7.png)

![image.png](/assets/blogImage/3994053-358e0b85dec95f1a.png)

![image.png](/assets/blogImage/3994053-e117da8394ea0169.png)

![image.png](/assets/blogImage/3994053-b6703baf65c2858c.png)

![image.png](/assets/blogImage/3994053-0e44f460d8fc63fa.png)

# 编译
```
mkdir build
cd build
#如果不想跑测试用例加上-DLLVM_INCLUDE_TESTS=OFF 
cmake -DCMAKE_BUILD_TYPE=Release -DLLVM_CREATE_XCODE_TOOLCHAIN=ON ../obfuscator/
make -j7
```
## 使用
这里原版提供了3种混淆方式分别是控制流扁平化,指令替换,虚假控制流程,用起来都是加cflags的方式。下面简单说下这几种模式。
### 控制流扁平化
这个模式主要是把一些if-else语句，嵌套成do-while语句

-mllvm -fla：激活控制流扁平化
-mllvm -split：激活基本块分割。在一起使用时改善展平。
-mllvm -split_num=3：如果激活了传递，则在每个基本块上应用3次。默认值：1

### 指令替换
这个模式主要用功能上等效但更复杂的指令序列替换标准二元运算符(+ , – , & , | 和 ^)

-mllvm -sub：激活指令替换
-mllvm -sub_loop=3：如果激活了传递，则在函数上应用3次。默认值：1
### 虚假控制流程
这个模式主要嵌套几层判断逻辑，一个简单的运算都会在外面包几层if-else，所以这个模式加上编译速度会慢很多因为要做几层假的逻辑包裹真正有用的代码。

另外说一下这个模式编译的时候要浪费相当长时间包哪几层不是闹得！

-mllvm -bcf：激活虚假控制流程
-mllvm -bcf_loop=3：如果激活了传递，则在函数上应用3次。默认值：1
-mllvm -bcf_prob=40：如果激活了传递，基本块将以40％的概率进行模糊处理。默认值：30
***
上面说完模式下面讲一下几种使用方式
### 直接用二进制文件
直接使用编译的二进制文件`build/bin/clang test.c -o test -mllvm -sub -mllvm -fla -mllvm -bcf`
### NDK集成
这里分为工具链的制作和项目里的配置。
#### 制作Toolchains
这里以修改最新的ndk r18为例，老的ndk版本比这更容易都在ndk-bundle/toolchains里放着需要修改的文件。
``` shell
#复制ndk的toolschain里的llvm
cp -r ndk-bundle/toolchains/llvm ndk-bundle/toolchains/ollvm
#删除prebuilt文件夹下的文件夹的bin和lib64，prebuilt文件夹下根据系统不同命名也不同
rm -rf ndk-bundle/toolchains/ollvm/prebuilt/darwin-x86_64/bin
rm -rf ndk-bundle/toolchains/ollvm/prebuilt/darwin-x86_64/lib64
#把我们之前编译好的ollvm下的bin和lib移到我们刚才删除bin和lib64的目录下
mv build/bin ndk-bundle/toolchains/ollvm/prebuilt/darwin-x86_64/
mv build/lib ndk-bundle/toolchains/ollvm/prebuilt/darwin-x86_64/
#复制ndk-bundle⁩/⁨build⁩/⁨core⁩/⁨toolchains的文件夹，这里根据自己对CPU架构的需求自己复制然后修改
cp -r ndk-bundle⁩/⁨build⁩/⁨core⁩/⁨toolchains/arm-linux-androideabi-clang⁩ ndk-bundle⁩/⁨build⁩/⁨core⁩/⁨toolchains/arm-linux-androideabi-clang-ollvm
```
最后把arm-linux-androideabi-clang-ollvm里的setup.mk文件进行修改

```
TOOLCHAIN_NAME := ollvm
TOOLCHAIN_ROOT := $(call get-toolchain-root,$(TOOLCHAIN_NAME))
TOOLCHAIN_PREFIX := $(TOOLCHAIN_ROOT)/bin
```
config.mk里是CPU架构,刚才是复制出来的所以不用修改，但如果要添加其他的自定义架构需要严格按照格式规范命名最初的文件夹，如mips的需要添加文件夹mipsel-linux-android-clang-ollvm，setup.mk和刚才的修改一样即可。
#### 项目中配置
到了项目里还需要修改两个文件：
在Android.mk 中添加混淆编译参数
```
LOCAL_CFLAGS += -mllvm -sub -mllvm -bcf -mllvm -fla
```
Application.mk中配置NDK_TOOLCHAIN_VERSION
```
#根据需要添加
APP_ABI := x86 armeabi-v7a x86_64 arm64-v8a mips armeabi mips64
#使用刚才我们做好的编译链
NDK_TOOLCHAIN_VERSION := ollvm
```

### Visual Studio集成
编译ollvm的时候，使用cmake-gui选择Visual Studio2015或者命令行选择`cmake -G "Visual Studio 14 2015" -DCMAKE_BUILD_TYPE=Release ../obfuscator/`
然后cmake会产生一个visual studio工程，用vs编译即可！
至于将Visual Studio的默认编译器换成clang编译，参考https://www.ishani.org/projects/ClangVSX/

Visual Studio2015起官方开始支持Clang，具体做法：
文件->新建->项目->已安装->Visual C++->跨平台->安装Clang with Microsoft CodeGen
Clang是一个完全不同的命令行工具链，这时候可以在工程配置中，平台工具集选项里找到Clang，然后使用ollvm的clang替换该clang即可。

### XCode集成
XCode里集成需要看版本，XCode10之前和之后是一个分水岭，XCode9之前和之后有一个小配置不同。
#### XCode10以前
```
$ cd /Applications/Xcode.app/Contents/PlugIns/Xcode3Core.ideplugin/Contents/SharedSupport/Developer/Library/Xcode/Plug-ins/
$ sudo cp -r Clang\ LLVM\ 1.0.xcplugin/ Obfuscator.xcplugin
$ cd Obfuscator.xcplugin/Contents/
$ sudo plutil -convert xml1 Info.plist
$ sudo vim Info.plist
```
修改:
```
<string>com.apple.compilers.clang</string> -> <string>com.apple.compilers.obfuscator</string>
<string>Clang LLVM 1.0 Compiler Xcode Plug-in</string> -> <string>Obfuscator Xcode Plug-in</string>
```
执行:
```
$ sudo plutil -convert binary1 Info.plist
$ cd Resources/
$ sudo mv Clang\ LLVM\ 1.0.xcspec Obfuscator.xcspec
$ sudo vim Obfuscator.xcspec
```
修改:
```
<key>Description</key>
<string>Apple LLVM 8.0 compiler</string> -> <string>Obfuscator 4.0 compiler</string>
<key>ExecPath</key>
<string>clang</string> -> <string>/path/to/obfuscator_bin/clang</string>
<key>Identifier</key>
<string>com.apple.compilers.llvm.clang.1_0</string> -> <string>com.apple.compilers.llvm.obfuscator.4_0</string>
<key>Name</key>
<string>Apple LLVM 8.0</string> -> <string>Obfuscator 4.0</string>
<key>Vendor</key>
<string>Apple</string> -> <string>HEIG-VD</string>
<key>Version</key>
<string>7.0</string> -> <string>4.0</string>
```
执行:
```
$ cd English.lproj/
$ sudo mv Apple\ LLVM\ 5.1.strings "Obfuscator 3.4.strings"
$ sudo plutil -convert xml1 Obfuscator\ 3.4.strings
$ sudo vim Obfuscator\ 3.4.strings 
```
修改:
```
<key>Description</key>
<string>Apple LLVM 8.0 compiler</string> -> <string>Obfuscator 4.0 compiler</string>
<key>Name</key>
<string>Apple LLVM 8.0</string> -> <string>Obfuscator 4.0</string>
<key>Vendor</key>
<string>Apple</string> -> <string>HEIG-VD</string>
<key>Version</key>
<string>7.0</string> -> <string>4.0</string>
```
执行:
```
$ sudo plutil -convert binary1 Obfuscator\ 3.4.strings
```
XCode9之后要设置`Enable Index-While-Building`成`NO`

![image.png](/assets/blogImage/3994053-07f0d5802141a7ae.png)

![image.png](/assets/blogImage/3994053-25ee66acdb89e0d2.png)

#### XCode10之后
xcode10之后无法使用添加ideplugin的方法，但添加编译链跑的依然可行，另外网上一些人说不能开bitcode，不能提交AppStore，用原版llvm改的ollvm的确有可能出现上述情况，所以我用苹果的swift-llvm改了一版暂时没去试着提交，或许可以，有兴趣的也可以自己下载使用试试[obfuscator](https://github.com/heroims/obfuscator/tree/swift-llvm-5.0)这版，特别备注由于修改没有针对swift部分所以用swift写的代码没混淆，回头有空的话再弄。

创建XCode的toolchain然后把生成的文件夹放到`/Library/Developer/`下
```
cd build
sudo make install-xcode-toolchain
mv /usr/local/Toolchains  /Library/Developer/
```
Toolchains下的.xctoolchain文件就是一个文件夹，进去修改info.plist
```
<key>CFBundleIdentifier</key>
<string>org.llvm.7.0.0svn</string> -> <string>org.ollvm-swift.5.0</string>
```
修改完在XCode的Toolchains下就会显示相应的名称

然后如图打开XCode选择Toolchaiins

![image.png](/assets/blogImage/3994053-110e3d513c3b7a6a.png)

![image.png](/assets/blogImage/3994053-9d09b7f7d4badfed.png)

![image.png](/assets/blogImage/3994053-b615d4cd5618c465.png)

按这些配置好后就算是可以用了。

# 最后
简单展示一下混淆后的成果

源码
![image.png](/assets/blogImage/3994053-09985d3c5f1764f0.png)

反编译未混淆代码
![image.png](/assets/blogImage/3994053-2624d222fb298087.png)

反编译混淆后代码
![image.png](/assets/blogImage/3994053-7137938cc9101b7c.png)



## 扩展：字符串混淆
原版是没有这功能的本来,[Armariris](https://github.com/GoSSIP-SJTU/Armariris) 提供了这个功能，我这也移植过来了，毕竟不难。
首先把`StringObfuscation`的.h,.cpp文件放到对应的`Obfuscation`文件夹下，然后分别修改下面的文件。
![image.png](/assets/blogImage/3994053-49e94a40169f202f.png)
### 用法
-mllvm -sobf：编译时候添加选项开启字符串加密
-mllvm -seed=0xdeadbeaf：指定随机数生成器种子
### 效果
看个添加了`-mllvm -sub -mllvm -sobf -mllvm -fla -mllvm -bcf`这么一串的效果。

源码

![image.png](/assets/blogImage/3994053-9d9e6f40bfb9aff4.png)

反编译未混淆代码
![image.png](/assets/blogImage/3994053-b86496d5219b2efd.png)

反编译混淆后代码

![image.png](/assets/blogImage/3994053-7851701440866c65.png)





