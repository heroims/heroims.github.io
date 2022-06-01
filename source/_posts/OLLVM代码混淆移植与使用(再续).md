---
title: OLLVM代码混淆移植与使用(再续)
date: 2022-04-09 23:09:05
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
# 前言
这波主要自己闲下来了，帮之前兼职区块链公司搞定了Android和iOS的BLE无网通信请了假，准备去自考本科最后四门，结果疫情太厉害直接取消了，本来还想着年中就可以申请毕业直接再准备一下考研请的是长假。。。结果计划永远赶不上变化，毕竟搞学历对现在的自己只是解个心结去个遗憾，对自己工作生活没啥用调整好情绪重新继续正经创业或工作吧，借着长假的时间思考思考人生，搞一搞之前没空处理的遗留项目。下面就直接进入主题把！

# 关键修改
### Legacy PM模式不生效
现在由于默认是NEW PM所以经常有人邮件我移植很完美编译也成功，就是没效果，这里做一下解答。主要两种方式解决，一种是在cmake的时候加一下`-DLLVM_ENABLE_NEW_PASS_MANAGER=OFF`来禁用掉NEW PM，这样在编译完成后使用的时候就可以了，还有一种就是走默认开启这，然后用ollvm编译自己项目时加上-flegacy-pass-manager的cflag，再加-mllvm原来哪些就可以正常使用了

### 14.0以后的修改
主要是`StringObfuscation.cpp`里面的两个地方，第一个是宏的修改编译时机，第二个就是CreateGEP,CreateLoad等多个方法需要传递指针类型了，原来不传会设置为null到了13.0里就开始内部通过对象获取类型，就像修改的这样，到了14.0干脆就是强制你必须传类型了。
![image.png](/assets/blogImage/1649574671473.jpg)
<!-- more -->
### 改为New Pass Manager
这个不好说修改了是好是坏，毕竟如果不修改，现在正常开启NEWPM编译完，利用-flegacy-pass-manager标识才能混淆，官方也说了在努力去除掉所有legacy pass manager，所以还是该准备一下。
这里只那两个Pass举例，因为改法都差不多，之所以是两个是因为一个是Function一个是Module,其实还有其他的好多种，只不过网上封装的混淆Pass也就用了这两种，想详细了解的，文档我反正没找到，但可以从源码中读逻辑。

#### FunctionAnalysisManager
首先以BogusControlFlow为例在`.h`里添加`#include "llvm/IR/PassManager.h"`的头文件，然后创建如下的Pass类
```
    class BogusControlFlowPass : public PassInfoMixin<BogusControlFlowPass>{ 
        public:
            PreservedAnalyses run(Function &F, FunctionAnalysisManager &AM);

            static bool isRequired() { return true; }
    };
```
在`.m`文件里先把之前初始化的方法包一层，为了在另一个Pass类里方便调用
![image.png](/assets/blogImage/WX20220411-141915@2x.png)
把下面的代码加完补上之前定义的类的方法就完成了
```
PreservedAnalyses BogusControlFlowPass::run(Function& F, FunctionAnalysisManager& AM) {
  BogusControlFlow bcf;
  if (bcf.runOnCustomFunction(F))
    return PreservedAnalyses::none();
  return PreservedAnalyses::all();
}
```

#### ModuleAnalysisManager
然后以StringObfuscation为例在`.h`里添加`#include "llvm/IR/PassManager.h"`的头文件，然后创建如下的Pass类
```
      class StringObfuscationPass : public PassInfoMixin<StringObfuscationPass>{ 
        public:
            PreservedAnalyses run(Module &M, ModuleAnalysisManager &AM);

            static bool isRequired() { return true; }
      };
```
在`.m`文件里比之前那个改动要多点，主要之前类就带了Pass后缀，如下直接去掉
![image.png](/assets/blogImage/WX20220411-143016@2x.png)
![image.png](/assets/blogImage/WX20220411-143058@2x.png)
再把之前初始化方法包一层，还是为了在另一个Pass类里方便调用
![image.png](/assets/blogImage/WX20220411-143722@2x.png)
把下面的代码加完补上之前定义的类的方法就完成了
```
PreservedAnalyses StringObfuscationPass::run(Module &M, ModuleAnalysisManager& AM) {
  StringObfuscation sop;
  if (sop.runCustomModule(M))
        return PreservedAnalyses::none();
  return PreservedAnalyses::all();
}
```

#### 注册 添加 Pass
把Pass类做完支持后就该添加和注册了，首先把头文件加到PassBuilder.cpp下，因为PassRegistry.def里注册Pass是从这边调用的。
![image.png](/assets/blogImage/WX20220411-144549@2x.png)
![image.png](/assets/blogImage/WX20220411-152632@2x.png)
然后找合适的时机插入Pass，与之前`PassManagerBuilder.cpp`对应的是`PassBuilderPipelines.cpp`，而方法则是`populateModulePassManager`对应`buildModuleOptimizationPipeline`,如下加到和之前类似的调用位置即可。
这里是头文件，和之前的一些处理，但基本没有用过
![image.png](/assets/blogImage/WX20220411-153711@2x.png)
下面是按之前位置挑了开头和结尾位置添加
![image.png](/assets/blogImage/WX20220411-153643@2x.png)

# Swift支持
想支持swift的混淆直接用我之前的swift分支是不现实的，因为苹果大量的修改想直接编译llvm就支持基本不可能，但如果直接选择编译swift的工具链来实现就简单的多，这个时候只需要下载[Swift](https://github.com/apple/swift)的源码，它在编译Toolchain时下载的llvm上把我之前的修改移植过来，然后编译出来就可以直接支持swift的混淆了。<font color='red'>(后来认真编译了几次，结果都是不报错混淆无效果，但整体思路应该没错下面写的没兴趣可以略过了。。。)</font>

根据[Xcode Wiki](https://en.wikipedia.org/wiki/Xcode)上的对应去切换git上的branch对应版本，有一点要注意通常你当前Xcode版本编译不了你当前Xcode用的，只能用上个版本编译你当前Xcode对应的，不过你也可以不管对应直接就用master的版本编译最新的。

官方给的编译教程在[这里](https://github.com/apple/swift/blob/main/docs/HowToGuides/GettingStarted.md),有兴趣的可以自己研磨，下面是我总结的。

这里我建议用ssh的方式，https的有时会出现连不上的情况，不加后面的`-with-ssh`走的就是https的。
```
git clone git@github.com:apple/swift.git
cd swift
utils/update-checkout --clone-with-ssh
```
![image](/assets/blogImage/WX20220410-211919@2x.png)
指令结束后swift同级目录会多出一堆当前master对应的依赖库，所以第一步`git clone`的时候一定要找个干净的目录。
然后切换到自己想要编译的版本，编译之前一定要看看自己Xcode版本
```
utils/update-checkout --scheme mybranchname
# OR
utils/update-checkout --tag mytagname
```

最后执行build_toolchain或build_script,其中build_toolchain最简单，傻瓜式编译，全自动，先看看有没有错，不报错就可以做ollvm移植了。
```
# 后面必须要跟个唯一标识
utils/build_toolchain com.xxxx
```

编译成功会如下图，多出两个文件夹两个tar.gz文件，里面都是.toolchain放`/Library/Developer/Toolchains`里Xcode就可以用了
![image](/assets/blogImage/WX20220411-214257@2x.png)


移植的话在`swift`同级目录有个`llvm-project`,这就是标准的llvm，之前怎么移植现在就怎么移植即可。比较简单的方式可以选择git patch文件或者找我swift-llvm-clang的分支用git cherry pick拉过去，这里的llvm都是指向的swift-llvm的。

下面提供了git的patch具体命令
```
cd ../llvm-project
# 下载patch文件
wget https://heroims.github.io/obfuscator/LegacyPass/ollvm14.patch
# 使用patch
git apply ollvm14.patch
```
如果失败，冲突了则用另一条命令如下，会生成冲突文件的对应.rej,直接看一下冲突的地方按提示修改即可。（大部分情况都会有冲突。。。）
```
git apply --reject --ignore-whitespace ollvm14.patch
```

把llvm移植完再用build_toolchain把工具链编译出来,这里再简单说一下build_toolchain其实内部就是在调用build_script。
```
./utils/build-script ${DRY_RUN} ${DISTCC_FLAG} ${PRESET_FILE_FLAGS} \
        ${SCCACHE_FLAG} \
        --preset="${PRESET_PREFIX}${SWIFT_PACKAGE}${NO_TEST}${USE_OS_RUNTIME}" \
        install_destdir="${SWIFT_INSTALL_DIR}" \
        installable_package="${SWIFT_INSTALLABLE_PACKAGE}" \
        install_toolchain_dir="${SWIFT_TOOLCHAIN_DIR}" \
        install_symroot="${SWIFT_INSTALL_SYMROOT}" \
        symbols_package="${SYMBOLS_PACKAGE}" \
        darwin_toolchain_bundle_identifier="${BUNDLE_IDENTIFIER}" \
        darwin_toolchain_display_name="${DISPLAY_NAME}" \
        darwin_toolchain_display_name_short="${DISPLAY_NAME_SHORT}" \
        darwin_toolchain_xctoolchain_name="${TOOLCHAIN_NAME}" \
        darwin_toolchain_version="${TOOLCHAIN_VERSION}" \
        darwin_toolchain_alias="Local" \
        darwin_toolchain_require_use_os_runtime="${REQUIRE_USE_OS_RUNTIME}"
```
如果想用build_toolchain做一些特殊配置也可以用--preset-file指定文件设置默认使用的是[build-presets.ini](https://github.com/apple/swift/blob/main/utils/build-presets.ini),里面用[preset: xxxx,xxx]来指定模块，而且还可以调用已有的，里面最基础的是[preset: mixin_osx_package_base]，仿照这可以写一个针对自己的llvm配置挑些自己需要的，我第一次直接用的master然后全量编译愣是直接编译了12个小时以上。。。。

# Android 轻量编译
之前我都是直接看一眼当前用的版本对应[google llvm](https://android.googlesource.com/toolchain/llvm-project)的版本就去下载切到对应commit位置，直接移植ollvm整体编译，后来看网上[有个套路](https://github.com/LeadroyaL/llvm-pass-tutorial)居然能利用ndk编译个Pass的.so文件直接搞定，所以记录一下。
对方也有些文章具体讲述[这里](https://www.leadroyal.cn/p/1008/)和[这里](https://xz.aliyun.com/t/6643)都有。

# 通用的轻量级编译
根据上面的套路，其实核心是Legacy Pass提供了自动注册功能，然后编译一个独立的动态加载的Pass，这样不需要对llvm本身进行修改，只是扩展一个模块，由此有了下边的代码分别对应Legacy Pass和New Pass两种模式的自动注册。

### Legacy Pass Manager
```
#include "Transforms/Obfuscation/BogusControlFlow.h"
#include "Transforms/Obfuscation/Flattening.h"
#include "Transforms/Obfuscation/Split.h"
#include "Transforms/Obfuscation/Substitution.h"
#include "Transforms/Obfuscation/StringObfuscation.h"
#include "llvm/IR/LegacyPassManager.h"
#include "llvm/Transforms/IPO/PassManagerBuilder.h"

using namespace llvm;

static void registerOllvmPass(const PassManagerBuilder &,
                              legacy::PassManagerBase &PM) {

    PM.add(createBogus(true));
#if LLVM_VERSION_MAJOR >= 9
    PM.add(createLowerSwitchPass());
#endif
    PM.add(createFlattening(true));
    PM.add(createSplitBasicBlock(true));
    PM.add(createSubstitution(true));
}

static void registerOllvmModulePass(const PassManagerBuilder &,
                              legacy::PassManagerBase &PM) {
    PM.add(createStringObfuscation(true));
}
static RegisterStandardPasses
        RegisterMyPass1(PassManagerBuilder::EP_EnabledOnOptLevel0,
                       registerOllvmModulePass);
static RegisterStandardPasses
        RegisterMyPass2(PassManagerBuilder::EP_OptimizerLast,
                        registerOllvmModulePass);
static RegisterStandardPasses
        RegisterMyPass3(PassManagerBuilder::EP_EarlyAsPossible,
                       registerOllvmFunctionPass);
```

### New Pass Manager
```
#include "Transforms/Obfuscation/BogusControlFlow.h"
#include "Transforms/Obfuscation/Flattening.h"
#include "Transforms/Obfuscation/Split.h"
#include "Transforms/Obfuscation/Substitution.h"
#include "Transforms/Obfuscation/StringObfuscation.h"
#include "llvm/Passes/PassBuilder.h"
#include "llvm/Passes/PassPlugin.h"

llvm::PassPluginLibraryInfo getOllvmPluginInfo() {
  return {
    LLVM_PLUGIN_API_VERSION, "OpcodeCounter", LLVM_VERSION_STRING,
        [](PassBuilder &PB) {

            // #1 注册标记 "opt -passes=obf-bcf"
            PB.registerPipelineParsingCallback(
              [&](StringRef Name, FunctionPassManager &FPM,
                  ArrayRef<PassBuilder::PipelineElement>) {
                if (Name == "obf-bcf") {
                  FPM.addPass(BogusControlFlowPass());
                  return true;
                }
                if(Name == "obf-fla"){
                  FPM.addPass(FlatteningPass());
                  return true;
                }
                if(Name == "obf-sub"){
                  FPM.addPass(SubstitutionPass());
                  return true;
                }
                if(Name == "obf-split"){
                  FPM.addPass(SplitBasicBlockPass());
                  return true;
                }
                return false;
              });
            PB.registerPipelineParsingCallback(
              [&](StringRef Name, ModulePassManager &MPM,
                  ArrayRef<PassBuilder::PipelineElement>) {
                if (Name == "obf-str") {
                  MPM.addPass(StringObfuscationPass());
                  return true;
                }
                return false;
              });

            // #2 找到具体时机插入pass
            //registerVectorizerStartEPCallback这个方法插入需要加-O1的flag不然可能不生效会被跳过
            PB.registerPipelineStartEPCallback(
              [](llvm::FunctionPassManager &PM,
                 llvm::PassBuilder::OptimizationLevel Level) {
                PM.addPass(SplitBasicBlockPass());
                PM.addPass(BogusControlFlowPass());
                #if LLVM_VERSION_MAJOR >= 9
                    PM.addPass(LowerSwitchPass());
                #endif
                PM.addPass(FlatteningPass());
                PM.addPass(SubstitutionPass());
                
              });
            PB.registerOptimizerLastEPCallback(
              [](llvm::ModulePassManager &PM,
                 llvm::PassBuilder::OptimizationLevel Level) {
                PM.addPass(StringObfuscationPass());
              });
          
        };
}

extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
  return getOllvmPluginInfo();
}
```
创建个PMRegistration.cpp,放到Obfuscation里再改一下CMakeLists.txt，执行完cmake构建好项目可以单独编译这个Obfuscation模块，其实用这个方法也就不需要再修改llvm本身了，移植起来方便多了只需要添加文件即可。用patch也就基本见不到冲突了。