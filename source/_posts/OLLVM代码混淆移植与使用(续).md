---
title: OLLVM代码混淆移植与使用(续)
date: 2021-11-09 23:09:05
tags:
    - LLVM
    - OLLVM
    - Clang
    - 混淆
    - XCode
    - NDK
    - Visual Studio
typora-root-url: ../
---

# 现状

随着时间时间推移，类似库都基本不维护了，毕竟LLVM新版本改动多还好说，再多只要看看关键位置就大差不差知道怎么改怎么兼容，最难的就是编译，每次编译调试不知不觉没干啥就能耗人一天，另外就是以前的人现在都不知道还在不在业内，回归正题，其实修改的地方并不多，下面我就整理了一下关键位置，主要就是类型和调用顺序变了一下，了解以后去修改类似的项目也是手到擒来。

# 关键修改 
### 9.0以后的修改
这里类型写法更加标准一步一步转
![image.png](/assets/blogImage/3994053-8d78f46f6c1acb2c.webp)
这里是9以后不在调用这个方法了，导致fla不生效，也可以在`Flattening.cpp`里面修改添加
![image.png](/assets/blogImage/3994053-8354e86d99016a71.webp)

<!-- more -->

### 10.0以后的修改

这里是LoadInst初始化多加了个类型参数，类似地方全改一遍
![image.png](/assets/blogImage/3994053-f4eded6ede7d355f.webp)
![image.png](/assets/blogImage/3994053-179bfa5cc5cac5ad.webp)
这里传入类型修改一下
![image.png](/assets/blogImage/3994053-2793ebe6fce228a3.webp)
再然后`CryptoUtils.h`里有一些宏定义是非常短的名称代表方法，这里再全局里容易有歧义，可以批量移到`CryptoUtils.cpp`里，因为只有这个类再用，其他地方也没有用这些短名称的宏方法
![image.png](/assets/blogImage/3994053-233a4ab906d1dad8.webp)
其他地方和原来一样不变。

### XCode12 以后
首先编译要多家几个项目，根据自己需要可以多填几个
```
cd build
cmake -DCMAKE_BUILD_TYPE=Release -DLLVM_CREATE_XCODE_TOOLCHAIN=ON -DLLVM_ENABLE_PROJECTS="clang;libcxx;libcxxabi" ../obfuscator/
make -j7
sudo make install-xcode-toolchain
mv /usr/local/Toolchains  /Library/Developer/
```
然后在Build Settings里找到`C++ Language Dialect` 和 `C++ Standard Library`使用默认是为了直接走编译链构建的版本，如果你还加了其他的也需要注意一下别的确保都走你编译的版本
![build3.png](/assets/blogImage/3994053-11c43091e54007da.webp)

