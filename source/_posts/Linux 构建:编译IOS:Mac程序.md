---
title: Linux 构建/编译IOS/Mac程序
date: 2017-09-10 02:09:05
tags:
    - IOS
    - 构建
    - 编译
    - Linux
    - 架构设计
---

# 前言
理论上这是一个很好的想法，但真正落实到实践，简直坑的不要不要的！笔者最终在CentOS上实在搞不动了，只在Ubuntu上弄了，关键CentOS是公司环境也不敢太造次。

整个东西写出来主要是让自己记住坑太多，不是实在想不开没事干，千万不要再搞。

思路来源https://github.com/facebook/xcbuild/issues/37
# 目的
做这个事情起初只是因为我司不提供Mac机器来做自动打包，但又想要让我做自动打包，能提供的也只有CentOS系统，不过很快就发现打包其实完全不可能，因为证书问题很难解决，然后想着跑跑CI的test也不错才继续研究了一下。。。。毕竟我们Model都是自动生成的，git提交后跑一下CI确保能各端运行正常还是很有用的，另外版本归档打tag的时候跑一下CocoaPods把完整可执行代码构建好拉下来归档也是挺不错的！

现实是恐怕就CocoaPods构建是比较稳定100%没问题，编译这个完全不敢说ok。。。所以有条件的话还是建议用有台Mac专门来干这些事，不然实在太麻烦，哪怕ssh到一台同事电脑悄悄开个账户搞都比在linux上搞靠谱！

如图实现，没mac和有mac简直就是天壤之别的恶心差距！

![Untitled.png](/assets/blogImage/3994053-fad34baa2afc46ac.png)

# 安装配置
## 准备工作
ruby （为了支持CocoaPods）
[clang](http://releases.llvm.org/download.html) 4.0或以上（为了支持IOS10，如果你准备的sdk版本不高，那就无所谓了，关键是对应上）
[Apple clang](https://opensource.apple.com/tarballs/clang/)(用他的话妥妥的手动apt-get拯救不了你)
[cctools-port](https://github.com/tpoechtrager/cctools-port.git) 生成ios工具链（ios-toolchain）回头替换掉Xcode里的
[ninja](https://github.com/ninja-build/ninja.git)
[xcbuild](https://github.com/facebook/xcbuild.git)
Xcode（确切的说其实只要三个文件夹但我很不放心用了整个）
## CocoaPods 安装
这里不得不提CocoaPods兼容性真心不错
CentOS
```
sudo yum install ruby
sudo gem install cocoapods
```
Ubuntu
```
sudo apt-get install ruby
sudo gem install cocoapods
```
就是如此简单！已经时可用状态了！
## Clang 安装
接下来就不细说CentOS了应为基本上大部分库都是要源码安装。。。
首先`clang --version`看看版本是不是自己想要，多半都不是然后直接删了
```
sudo apt-get autoremove clang
```
然后搞一下软件源`/etc/apt/sources.list`不然各种依赖包各种找不到让你爽歪歪
```
#deb cdrom:[Ubuntu 16.04.2 LTS _Xenial Xerus_ - Release amd64 (20170215.2)]/ xenial main restricted
# See http://help.ubuntu.com/community/UpgradeNotes for how to upgrade to
# newer versions of the distribution.
deb http://us.archive.ubuntu.com/ubuntu/ xenial main restricted
# deb-src http://us.archive.ubuntu.com/ubuntu/ xenial main restricted
## Major bug fix updates produced after the final release of the
## distribution.
deb http://us.archive.ubuntu.com/ubuntu/ xenial-updates main restricted
# deb-src http://us.archive.ubuntu.com/ubuntu/ xenial-updates main restricted
## N.B. software from this repository is ENTIRELY UNSUPPORTED by the Ubuntu
## team. Also, please note that software in universe WILL NOT receive any
## review or updates from the Ubuntu security team.
deb http://us.archive.ubuntu.com/ubuntu/ xenial universe
# deb-src http://us.archive.ubuntu.com/ubuntu/ xenial universe
deb http://us.archive.ubuntu.com/ubuntu/ xenial-updates universe
# deb-src http://us.archive.ubuntu.com/ubuntu/ xenial-updates universe
## N.B. software from this repository is ENTIRELY UNSUPPORTED by the Ubuntu 
## team, and may not be under a free licence. Please satisfy yourself as to 
## your rights to use the software. Also, please note that software in 
## multiverse WILL NOT receive any review or updates from the Ubuntu
## security team.
deb http://us.archive.ubuntu.com/ubuntu/ xenial multiverse
# deb-src http://us.archive.ubuntu.com/ubuntu/ xenial multiverse
deb http://us.archive.ubuntu.com/ubuntu/ xenial-updates multiverse
# deb-src http://us.archive.ubuntu.com/ubuntu/ xenial-updates multiverse
## N.B. software from this repository may not have been tested as
## extensively as that contained in the main release, although it includes
## newer versions of some applications which may provide useful features.
## Also, please note that software in backports WILL NOT receive any review
## or updates from the Ubuntu security team.
deb http://us.archive.ubuntu.com/ubuntu/ xenial-backports main restricted universe multiverse
# deb-src http://us.archive.ubuntu.com/ubuntu/ xenial-backports main restricted universe multiverse
## Uncomment the following two lines to add software from Canonical's
## 'partner' repository.
## This software is not part of Ubuntu, but is offered by Canonical and the
## respective vendors as a service to Ubuntu users.
# deb http://archive.canonical.com/ubuntu xenial partner
# deb-src http://archive.canonical.com/ubuntu xenial partner
deb http://security.ubuntu.com/ubuntu xenial-security main restricted
# deb-src http://security.ubuntu.com/ubuntu xenial-security main restricted
deb http://security.ubuntu.com/ubuntu xenial-security universe
# deb-src http://security.ubuntu.com/ubuntu xenial-security universe
deb http://security.ubuntu.com/ubuntu xenial-security multiverse
# deb-src http://security.ubuntu.com/ubuntu xenial-security multiverse
```
下面大更新一波，安装
```
sudo apt-get update
sudo apt-get install git gcc cmake libssl-dev libtool autoconf automake clang-4.0
```
除了错误别找我，直接谷歌去找源
安装完`clang`看看位置如果不是`usr/bin`
执行下面命令制作软链接，只为保险不是必需的。。。。
```
sudo ln -s /usr/bin/clang-4.0 /usr/bin/clang
sudo ln -s /usr/bin/clang++-4.0 /usr/bin/clang++
```
吐槽一波CentOS你更新了源也没用该没有的还是没有建议直接源码安装。

```
#下载llvm源码
wget http://llvm.org/releases/4.0.1/llvm-4.0.1.src.tar.xz
tar xf llvm-4.0.1.src.tar.xz
mv llvm-4.0.1.src llvm

#下载clang源码
cd llvm/tools
wget http://llvm.org/releases/4.0.1/cfe-4.0.1.src.tar.xz
tar xf cfe-4.0.1.src.tar.xz
mv cfe-4.0.1.src clang
cd ../..

#下载clang-tools-extra源码  可选
cd llvm/tools/clang/tools
wget http://llvm.org/releases/4.0.1/clang-tools-extra-4.0.1.src.tar.xz
tar xf clang-tools-extra-4.0.1.src.tar.xz
mv clang-tools-extra-4.0.1.src  extra
cd ../../../..

#下载compiler-rt源码 可选
cd llvm/projects
wget http://llvm.org/releases/4.0.1/compiler-rt-4.0.1.src.tar.xz
tar xf compiler-rt-4.0.1.src.tar.xz
mv compiler-rt-4.0.1.src compiler-rt
cd ../..

mkdir llvmbuild
cd llvmbuild
#正常套路安装
#设置配置
#–prefix=directory — 设置llvm编译的安装路径(default/usr/local). 
#–enable-optimized — 是否选择优化(defaultis NO)，yes是指安装一个Release版本. 
#–enable-assertions — 是否断言检查(default is YES).
../llvm/configure --enable-optimized --enable-targets=host-only --prefix=/usr/bin
#构建
cmake ../llvm
make
#安装
sudo make install

#ninja套路安装
#设置配置
cmake -G Ninja -DCMAKE_INSTALL_PREFIX=/usr/bin -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_FFI=ON -DLLVM_BUILD_LLVM_DYLIB=ON -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DLLVM_TARGETS_TO_BUILD="host" -Wno-dev ../llvm
#构建想几核自己根据配置设置
ninja -j4
#安装
ninja install
```
## Apple Clang 安装
步骤如上面CentOS源码安装Clang一样，只是把Clang地址改了下
```
cd llvm/tools
wget https://opensource.apple.com/tarballs/clang/clang-800.0.42.1.tar.gz
tar xf clang-800.0.42.1.tar.gz
mv clang-800.0.42.1 clang
cd ../..
```

## cctools-port 制作ios-toolchain 的Clang
### ios sdk 打包
下载完Xcode直接执行下面命令
```

SDK=$(ls -l Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs | grep " -> iPhoneOS.sdk" | head -n1 | awk '{print $9}')
cp -r Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk /tmp/$SDK 1>/dev/null
cp -r Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/include/c++/v1 /tmp/$SDK/usr/include/c++ 1>/dev/null
pushd /tmp
tar -cvzf $SDK.tar.gz $SDK
rm -rf $SDK
mv $SDK.tar.gz ~
popd
```
### 制作 iOS armv7 工具链
```
cd cctools-port
IPHONEOS_DEPLOYMENT_TARGET=5.0 usage_examples/ios_toolchain/build.sh ~/iPhoneOS10.0.sdk.tar.gz armv7
```
制作工具链成功后会提示*** all done ***

将生成的工具链移到 /usr/local/ 目录并更名为 ios-armv7
```
sudo mv usage_examples/ios_toolchain/target /usr/local/ios-armv7
```
将库文件拷贝一份，放进公共库 /usr/lib
```
sudo cp /usr/local/ios-armv7/lib/libtapi.so /usr/lib
```

最后将工具链的 bin 目录加入PATH，方便调用
```
export PATH=$PATH:/usr/local/ios-armv7/bin
```
### 制作 iOS arm64 工具链
```
cd cctools-port
IPHONEOS_DEPLOYMENT_TARGET=5.0 usage_examples/ios_toolchain/build.sh ~/iPhoneOS10.0.sdk.tar.gz arm64
```
制作工具链成功后会提示*** all done ***

将生成的工具链  /usr/local/ 目录并更名为 ios-arm64
```
sudo mv usage_examples/ios_toolchain/target /usr/local/ios-arm64
```
使用 rename 命令重命名前缀以与 armv7 区分开来把 arm- 前缀改为 aarch64- 前缀
```
rename 's/arm-/aarch64-/' /usr/local/ios-arm64/bin/*
sudo rm /usr/local/ios-arm64/aarch64-apple-darwin11-clang++
sudo ln -s /usr/local/ios-arm64/aarch64-apple-darwin11-clang /usr/local/ios-arm64/aarch64-apple-darwin11-clang++
```
将库文件拷贝一份，放进公共库 /usr/lib
```
sudo cp /usr/local/ios-arm64/lib/libtapi.so /usr/lib
```
最后将工具链的 bin 目录加入PATH，方便调用
```
export PATH=$PATH:/usr/local/ios-arm64/bin
```

其实到这基本完活了，单跑.m文件已经没问题了，下面只是为了更加方便的编译项目

这里已经可以跑个测试程序编译试试了如`arrch-apple-darwin11-clang helloworld.c -o helloworld`或`arm-apple-darwin11-clang helloworld.c -o helloworld`

### 合并 armv7 和 arm64 工具链(可选)
一般用`ar`命令做合并不过容易出问题

### 替换xctoolchain
把最后想要用的生成的工具链clang 和 clang++文件替换到`usr/bin`下（不替换也可以达成目的即可），相当于回头编译的时候走我们产出的这套clang环境。总之就是思路就是让`xcbuild`走我们的这套clang去打包编译程序即可。如果你用Apple Clang或直接Clang能打包编译也就不用自己手动做这个事了。


## Ninja 安装
Ubuntu一行命令解决。。。。
```
apt-get install ninja-build
```
CentOS天坑模式开启不出意外先升级cmake
源码安装最新版[cmake](https://gitlab.kitware.com/cmake/cmake.git)解压执行`./bootstrap && make && make install` 就ok了，3个命令合一。
然后下载ninja源码
```
git clone git://github.com/ninja-build/ninja.git && cd ninja
git checkout release
./configure.py --bootstrap
```
最后把Path加一下
```
export PATH=$PATH:/xxx/xxx/ninja
```
## xcbuild 安装
```
git clone https://github.com/facebook/xcbuild
cd xcbuild
git submodule update --init
make
```
最后把Path加一下和DEVELOPER_DIR
DEVELOPER_DIR就是Xcode的目录，Xcode其实只需要有三个文件夹`Xcode.app/Contents/PlugIns`，`Xcode.app/Contents/Developer/Toolchains`,`Xcode.app/Contents/Developer/Platforms`
```
export PATH=$PATH:/xxx/xxx/xcbuild/build
export DEVELOPER_DIR=/xxx/xxx/Xcode.app
```
# 使用
完全可以像在Mac下一样用没有任何区别，不过也就CocoaPods完全ok，xcbuild指令和xcodebuild用法完全一样直接用xcodebuild也没问题，只不过能不能编译成功就听天由命吧！坑很深，慎入！
