---
title: 关于LLVM，这些东西你必须要知道

date: 2016-12-29

author:  test
url: www.baidu.com

category: ios
list_number: true 
tags: 
- LLVM

---
只要你和代码打交道，了解编译器的工作流程和原理定会让你受益无穷，无论是分析程序，还是基于它写自己的插件，甚至学习一门全新的语音。通过本文，将带你了解LLVM，并使用LLVM来完成一些有意思的事情。<!-- more -->

## 1. 什么是LLVM？

>The LLVM Project is a collection of modular and reusable compiler and toolchain technologies.

简单来说，LLVM项目是一系列分模块、可重用的编译工具链。它提供了一种代码编写良好的中间表示(IR)，可以作为多种语言的后端，还可以提供与变成语言无关的优化和针对多种cpu的代码生成功能。

先来看下LLVM架构的主要组成部分：

* 前端：前端用来获取源代码然后将它转变为某种中间表示，我们可以选择不同的编译器来作为LLVM的前端，如gcc，clang。
* Pass(通常翻译为“流程”)：Pass用来将程序的中间表示之间相互变换。一般情况下，Pass可以用来优化代码，这部分通常是我们关注的部分。
* 后端：后端用来生成实际的机器码。

虽然如今大多数编译器都采用的是这种架构，但是LLVM不同的就是对于不同的语言它都提供了同一种中间表示。传统的编译器的架构如下:

<center><img src="http://7xtdl4.com1.z0.glb.clouddn.com/script_1482136450962.png"/></center>

LLVM的架构如下：

<center><img src="http://7xtdl4.com1.z0.glb.clouddn.com/script_1482136601642.png"/></center>

当编译器需要支持多种源代码和目标架构时，基于LLVM的架构，设计一门新的语言只需要去实现一个新的前端就行了，支持新的后端架构也只需要实现一个新的后端就行了。其它部分完成可以复用，就不用再重新设计一次了。

## 2. 安装编译LLVM

这里使用clang作为前端:

1. 直接从官网下载:[http://releases.llvm.org/download.html
](http://releases.llvm.org/download.html)

2. svn获取

```
svn co http://llvm.org/svn/llvm-project/llvm/trunk llvm
cd llvm/tools
svn co http://llvm.org/svn/llvm-project/cfe/trunk clang
cd ../projects
svn co http://llvm.org/svn/llvm-project/compiler-rt/trunk compiler-rt
cd ../tools/clang/tools
svn co http://llvm.org/svn/llvm-project/clang-tools-extra/trunk extra
```

3. git获取

```
git clone http://llvm.org/git/llvm.git
cd llvm/tools
git clone http://llvm.org/git/clang.git
cd ../projects
git clone http://llvm.org/git/compiler-rt.git
cd ../tools/clang/tools
git clone http://llvm.org/git/clang-tools-extra.git
```

最新的LLVM只支持cmake来编译了，首先安装cmake。

```
brew install cmake
```

编译：

```
mkdir build
cmake /path/to/llvm/source
cmake --build .
```

编译时间比较长，而且编译结果会生成20G左右的文件。

编译完成后，就能在`build/bin/`目录下面找到生成的工具了。

## 3. 从源码到可执行文件

我们在开发的时候的时候，如果想要生成一个可执行文件或应用，我们点击run就完事了，那么在点击run之后编译器背后又做了哪些事情呢？

我们先来一个例子：

```
#import <Foundation/Foundation.h>

#define TEN 10

int main(){
    @autoreleasepool {
        int numberOne = TEN;
        int numberTwo = 8;
        NSString* name = [[NSString alloc] initWithUTF8String:"AloneMonkey"];
        int age = numberOne + numberTwo;
        NSLog(@"Hello, %@, Age: %d", name, age);
    }
    return 0;
}
```

上面这个文件，我们可以通过命令行直接编译，然后链接：

```
xcrun -sdk iphoneos clang -arch armv7 -F Foundation -fobjc-arc -c main.m -o main.o
xcrun -sdk iphoneos clang main.o -arch armv7 -fobjc-arc -framework Foundation -o main
```

拷贝到手机运行：

```
monkeyde-iPhone:/tmp root# ./main
2016-12-19 17:16:34.654 main[2164:213100] Hello, AloneMonkey, Age: 18
```

大家不会以为就这样就完了吧，当然不是，我们要继续深入剖析。

### 3.1 预处理（Preprocess）

这部分包括macro宏的展开，import/include头文件的导入，以及#if等处理。


可以通过执行以下命令，来告诉clang只执行到预处理这一步：

```
clang -E main.m
```

执行完这个命令之后，我们会发现导入了很多的头文件内容。

```
......
# 1 "/System/Library/Frameworks/Foundation.framework/Headers/FoundationLegacySwiftCompatibility.h" 1 3
# 185 "/System/Library/Frameworks/Foundation.framework/Headers/Foundation.h" 2 3
# 2 "main.m" 2

int main(){
    @autoreleasepool {
        int numberOne = 10;
        int numberTwo = 8;
        NSString* name = [[NSString alloc] initWithUTF8String:"AloneMonkey"];
        int age = numberOne + numberTwo;
        NSLog(@"Hello, %@, Age: %d", name, age);
    }
    return 0;
}
```

可以看到上面的预处理已经把宏替换了，并且导入了头文件。但是这样的话会引入很多不会去改变的系统库比如Foundation，所以有了pch预处理文件，可以在这里去引入一些通用的头文件。

后来Xcode新建的项目里面去掉了pch文件，引入了moduels的概念，把一些通用的库打成modules的形式，然后导入，默认会加上-fmodules参数。

```
clang -E -fmodules main.m
```

这样的话，只需要@import一下就能导入对应库的modules模块了。

```
@import Foundation; 
int main(){
    @autoreleasepool {
        int numberOne = 10;
        int numberTwo = 8;
        NSString* name = [[NSString alloc] initWithUTF8String:"AloneMonkey"];
        int age = numberOne + numberTwo;
        NSLog(@"Hello, %@, Age: %d", name, age);
    }
    return 0;
}
```

### 3.2 词法分析 (Lexical Analysis)

在预处理之后，就要进行词法分析了，将预处理过的代码转化成一个个Token，比如左括号、右括号、等于、字符串等等。


```
clang -fmodules -fsyntax-only -Xclang -dump-tokens main.m
```

![image](http://7xtdl4.com1.z0.glb.clouddn.com/script_1482994028068.png)

### 3.3 语法分析 (Semantic Analysis)

根据当前语言的语法，验证语法是否正确，并将所有节点组合成抽象语法树(AST)

```
clang -fmodules -fsyntax-only -Xclang -ast-dump main.m
```

![image](http://7xtdl4.com1.z0.glb.clouddn.com/script_1482994071651.png)

语法树直观图:

![image](http://7xtdl4.com1.z0.glb.clouddn.com/script_1482150626825.png)


### 3.4 IR代码生成 (CodeGen)

CodeGen负责将语法树从顶至下遍历，翻译成LLVM IR，LLVM IR是Frontend的输出，也是LLVM Backerend的输入，桥接前后端。

可以在中间代码层次去做一些优化工作，我们在Xcode的编译设置里面也可以设置优化级别`-O1`,`-O3`,`-Os`。 还可以去写一些自己的Pass，这里需要解释一下什么是Pass。

Pass就是LLVM系统转化和优化的工作的一个节点，每个节点做一些工作，这些工作加起来就构成了LLVM整个系统的优化和转化。

```
clang -S -fobjc-arc -emit-llvm main.m -o main.ll
```

![image](http://7xtdl4.com1.z0.glb.clouddn.com/script_1482994093995.png)


### 3.5 生成字节码 (LLVM Bitcode)

我们在Xcode7中默认生成bitcode就是这种的中间形式存在， 开启了bitcode，那么苹果后台拿到的就是这种中间代码，苹果可以对bitcode做一个进一步的优化，如果有新的后端架构，仍然可以用这份bitcode去生成。

```
clang -emit-llvm -c main.m -o main.bc
```

![image](http://7xtdl4.com1.z0.glb.clouddn.com/script_1482218230417.png)

### 生成相关汇编

```
clang -S -fobjc-arc main.m -o main.s
```

![image](http://7xtdl4.com1.z0.glb.clouddn.com/script_1482994171109.png)


### 生成目标文件

```
clang -fmodules -c main.m -o main.o
```

![image](http://7xtdl4.com1.z0.glb.clouddn.com/script_1482218636504.png)

### 生成可执行文件

```
clang main.o -o main
./main
```

```
2016-12-20 15:25:42.299 main[8941:327306] Hello, AloneMonkey, Age: 18
```

### 整体流程

![image](http://7xtdl4.com1.z0.glb.clouddn.com/script_1482219005958.png)

## 可以用Clang做什么？

### libclang进行语法分析

可以使用libclang里面提供的方法对源文件进行语法分析，分析它的语法树，遍历语法树上面的每一个节点。可以用于检查拼写错误，或者做字符串加密。

来看一段代码的使用：

```
void *hand = dlopen("/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/libclang.dylib",RTLD_LAZY);
        
//初始化函数指针
initlibfunclist(hand);

CXIndex cxindex = myclang_createIndex(1, 1);

const char *filename = "/path/to/filename";

int index = 0;

const char ** new_command = malloc(10240);

NSMutableString *mus = [NSMutableString stringWithString:@"/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang -x objective-c -arch armv7 -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk"]; 

NSArray *arr = [mus componentsSeparatedByString:@" "];

for (NSString *tmp in arr) {
    new_command[index++] = [tmp UTF8String];
}

nameArr = [[NSMutableArray alloc] initWithCapacity:10];

TU = myclang_parseTranslationUnit(cxindex, filename, new_command, index, NULL, 0, myclang_defaultEditingTranslationUnitOptions());

CXCursor rootCursor = myclang_getTranslationUnitCursor(TU);

myclang_visitChildren(rootCursor, printVisitor, NULL);

myclang_disposeTranslationUnit(TU);
myclang_disposeIndex(cxindex);
free(new_command);

dlclose(hand);
```

然后我们就可以在`printVisitor`这个函数里面去遍历输入文件的语法树了。

![image](http://7xtdl4.com1.z0.glb.clouddn.com/script_1482994195691.png)


我们也通过通过python去调用用clang：

```
pip install clang
```

![image](http://7xtdl4.com1.z0.glb.clouddn.com/script_1482994863704.png)

![image](http://7xtdl4.com1.z0.glb.clouddn.com/script_1482994879267.png)

那么基于语法树的分析，我们可以针对字符串做加密：

<center><img src="http://7xtdl4.com1.z0.glb.clouddn.com/script_1482320827975.png"/></center>

从左上角的明文字符串，处理成右下角的介个样子~

### LibTooling

对语法树有完全的控制权，可以作为一个单独的命令使用，如：`clang-format`

```
clang-format main.m
```

我们也可以自己写一个这样的工具去遍历、访问、甚至修改语法树。 目录:`llvm/tools/clang/tools`

![image](http://7xtdl4.com1.z0.glb.clouddn.com/script_1482994222169.png)

上面的代码通过遍历语法树，去修改里面的方法名和返回变量名：

```
before:
void do_math(int *x) {
    *x += 5;
}

int main(void) {
    int result = -1, val = 4;
    do_math(&val);
    return result;
}

after:
** Rewrote function def: do_math
** Rewrote function call
** Rewrote ReturnStmt

Found 2 functions.

void add5(int *x) {
    *x += 5;
}

int main(void) {
    int result = -1, val = 4;
    add5(&val);
    return val;
}
```

那么，我们看到`LibTooling`对代码的语法树有完全的控制，那么我们可以基于它去检查命名的规范，甚至做一个代码的转换，比如实现OC转Swift。

### ClangPlugin

对语法树有完全的控制权，作为插件注入到编译流程中，可以影响build和决定编译过程。目录:`llvm/tools/clang/examples`

```
#include "clang/Driver/Options.h"
#include "clang/AST/AST.h"
#include "clang/AST/ASTContext.h"
#include "clang/AST/ASTConsumer.h"
#include "clang/AST/RecursiveASTVisitor.h"
#include "clang/Frontend/ASTConsumers.h"
#include "clang/Frontend/FrontendActions.h"
#include "clang/Frontend/CompilerInstance.h"
#include "clang/Frontend/FrontendPluginRegistry.h"
#include "clang/Rewrite/Core/Rewriter.h"

using namespace std;
using namespace clang;
using namespace llvm;

Rewriter rewriter;
int numFunctions = 0;


class ExampleVisitor : public RecursiveASTVisitor<ExampleVisitor> {
private:
    ASTContext *astContext; // used for getting additional AST info

public:
    explicit ExampleVisitor(CompilerInstance *CI) 
      : astContext(&(CI->getASTContext())) // initialize private members
    {
        rewriter.setSourceMgr(astContext->getSourceManager(), astContext->getLangOpts());
    }

    virtual bool VisitFunctionDecl(FunctionDecl *func) {
        numFunctions++;
        string funcName = func->getNameInfo().getName().getAsString();
        if (funcName == "do_math") {
            rewriter.ReplaceText(func->getLocation(), funcName.length(), "add5");
            errs() << "** Rewrote function def: " << funcName << "\n";
        }    
        return true;
    }

    virtual bool VisitStmt(Stmt *st) {
        if (ReturnStmt *ret = dyn_cast<ReturnStmt>(st)) {
            rewriter.ReplaceText(ret->getRetValue()->getLocStart(), 6, "val");
            errs() << "** Rewrote ReturnStmt\n";
        }        
        if (CallExpr *call = dyn_cast<CallExpr>(st)) {
            rewriter.ReplaceText(call->getLocStart(), 7, "add5");
            errs() << "** Rewrote function call\n";
        }
        return true;
    }
};



class ExampleASTConsumer : public ASTConsumer {
private:
    ExampleVisitor *visitor; // doesn't have to be private

public:
    // override the constructor in order to pass CI
    explicit ExampleASTConsumer(CompilerInstance *CI):
        visitor(new ExampleVisitor(CI)) { } // initialize the visitor

    // override this to call our ExampleVisitor on the entire source file
    virtual void HandleTranslationUnit(ASTContext &Context) {
        /* we can use ASTContext to get the TranslationUnitDecl, which is
             a single Decl that collectively represents the entire source file */
        visitor->TraverseDecl(Context.getTranslationUnitDecl());
    }
};

class PluginExampleAction : public PluginASTAction {
protected:
    // this gets called by Clang when it invokes our Plugin
    // Note that unique pointer is used here.
    std::unique_ptr<ASTConsumer> CreateASTConsumer(CompilerInstance &CI, StringRef file) {
        return llvm::make_unique<ExampleASTConsumer>(&CI);
    }

    // implement this function if you want to parse custom cmd-line args
    bool ParseArgs(const CompilerInstance &CI, const vector<string> &args) {
        return true;
    }
};


static FrontendPluginRegistry::Add<PluginExampleAction> X("-example-plugin", "simple Plugin example");
```

```
clang -Xclang -load -Xclang ../build/lib/PluginExample.dylib -Xclang -plugin -Xclang -example-plugin -c testPlugin.c

** Rewrote function def: do_math
** Rewrote function call
** Rewrote ReturnStmt
```

我们可以基于ClangPlugin做些什么事情呢？我们可以用来定义一些编码规范，比如代码风格检查，命名检查等等。下面是我写的判断类名前两个字母是不是大写的例子，如果不是报错。(当然这只是一个例子而已。。。)

![image](http://7xtdl4.com1.z0.glb.clouddn.com/script_1482318703701.png)

## 动手写Pass

### 一个简单的Pass

前面我们说到，Pass就是LLVM系统转化和优化的工作的一个节点，当然我们也可以写一个这样的节点去做一些自己的优化工作或者其它的操作。下面我们来看一下一个简单Pass的编写流程：

1.创建头文件  

```
cd llvm/include/llvm/Transforms/
mkdir Obfuscation
cd Obfuscation
touch SimplePass.h
```

写入内容：

```
#include "llvm/IR/Function.h"
#include "llvm/Pass.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/IR/Intrinsics.h"
#include "llvm/IR/Instructions.h"
#include "llvm/IR/LegacyPassManager.h"
#include "llvm/Transforms/IPO/PassManagerBuilder.h"

// Namespace
using namespace std;

namespace llvm {
	Pass *createSimplePass(bool flag);
}
```

2.创建源文件

```
cd llvm/lib/Transforms/
mkdir Obfuscation
cd Obfuscation

touch CMakeLists.txt
touch LLVMBuild.txt
touch SimplePass.cpp
```

CMakeLists.txt:

```
add_llvm_loadable_module(LLVMObfuscation
  SimplePass.cpp
  
  )

  add_dependencies(LLVMObfuscation intrinsics_gen)
```

LLVMBuild.txt:

```
[component_0]
type = Library
name = Obfuscation
parent = Transforms
library_name = Obfuscation
```

SimplePass.cpp:

```
#include "llvm/Transforms/Obfuscation/SimplePass.h"

using namespace llvm;

namespace {
    struct SimplePass : public FunctionPass {
        static char ID; // Pass identification, replacement for typeid
        bool flag;
         
        SimplePass() : FunctionPass(ID) {}
        SimplePass(bool flag) : FunctionPass(ID) {
        	this->flag = flag;
        }
         
        bool runOnFunction(Function &F) override {
        	if(this->flag){
                Function *tmp = &F;
                // 遍历函数中的所有基本块
                for (Function::iterator bb = tmp->begin(); bb != tmp->end(); ++bb) {
                    // 遍历基本块中的每条指令
                    for (BasicBlock::iterator inst = bb->begin(); inst != bb->end(); ++inst) {
                        // 是否是add指令
                        if (inst->isBinaryOp()) {
                            if (inst->getOpcode() == Instruction::Add) {
                                ob_add(cast<BinaryOperator>(inst));
                            }
                        }
                    }
                }
            }
            return false;
        }
         
        // a+b === a-(-b)
        void ob_add(BinaryOperator *bo) {
            BinaryOperator *op = NULL;
             
            if (bo->getOpcode() == Instruction::Add) {
                // 生成 (－b)
                op = BinaryOperator::CreateNeg(bo->getOperand(1), "", bo);
                // 生成 a-(-b)
                op = BinaryOperator::Create(Instruction::Sub, bo->getOperand(0), op, "", bo);
                 
                op->setHasNoSignedWrap(bo->hasNoSignedWrap());
                op->setHasNoUnsignedWrap(bo->hasNoUnsignedWrap());
            }
             
            // 替换所有出现该指令的地方
            bo->replaceAllUsesWith(op);
        }
    };
}
 
char SimplePass::ID = 0;
 
// 注册pass 命令行选项显示为simplepass
static RegisterPass<SimplePass> X("simplepass", "this is a Simple Pass");
Pass *llvm::createSimplePass() { return new SimplePass(); }
```

修改`.../Transforms/LLVMBuild.txt`, 加上刚刚写的模块`Obfuscation`

```
subdirectories = Coroutines IPO InstCombine Instrumentation Scalar Utils Vectorize ObjCARC Obfuscation
```
修改`.../Transforms/CMakeLists.txt`,  加上刚刚写的模块`Obfuscation`

```
add_subdirectory(Obfuscation)
```

编译生成：`LLVMSimplePass.dylib`

因为Pass是作用于中间代码，所以我们首先要生成一份中间代码：

```
clang -emit-llvm -c test.c -o test.bc
```

然后加载Pass优化：

```
../build/bin/opt -load ../build/lib/LLVMSimplePass.dylib -test < test.bc > after_test.bc
```

对比中间代码：

```
llvm-dis test.bc -o test.ll
llvm-dis after_test.bc -o after_test.ll
```

```
test.ll
......
entry:
  %retval = alloca i32, align 4
  %a = alloca i32, align 4
  %b = alloca i32, align 4
  %c = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 3, i32* %a, align 4
  store i32 4, i32* %b, align 4
  %0 = load i32, i32* %a, align 4
  %1 = load i32, i32* %b, align 4
  %add = add nsw i32 %0, %1
  store i32 %add, i32* %c, align 4
  %2 = load i32, i32* %c, align 4
  %call = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([4 x i8], [4 x i8]* @.str, i32 0, i32 0), i32 %2)
  ret i32 0
}
......
```

```
after_test.ll
......
entry:
  %retval = alloca i32, align 4
  %a = alloca i32, align 4
  %b = alloca i32, align 4
  %c = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 3, i32* %a, align 4
  store i32 4, i32* %b, align 4
  %0 = load i32, i32* %a, align 4
  %1 = load i32, i32* %b, align 4
  %2 = sub i32 0, %1
  %3 = sub nsw i32 %0, %2
  %add = add nsw i32 %0, %1
  store i32 %3, i32* %c, align 4
  %4 = load i32, i32* %c, align 4
  %call = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([4 x i8], [4 x i8]* @.str, i32 0, i32 0), i32 %4)
  ret i32 0
}
......
```

这里写的Pass只是把a+b简单的替换成了a-(-b),只是一个演示，怎么去写自己的Pass，并且作用于代码。

### 将Pass加入PassManager管理

上面我们是单独去加载Pass动态库，这里我们将Pass加入PassManager，这样我们就可以直接通过clang的参数去加载我们的Pass了。

首先在`llvm/lib/Transforms/IPO/PassManagerBuilder.cpp`添加头文件。

```
#include "llvm/Transforms/Obfuscation/SimplePass.h"
```

然后添加如下语句：

```
static cl::opt<bool> SimplePass("simplepass", cl::init(false),
                           cl::desc("Enable simple pass"));
```

然后在`populateModulePassManager`这个函数中添加如下代码：

```
MPM.add(createSimplePass(SimplePass));
```

最后在IPO这个目录的`LLVMBuild.txt`中添加库的支持，否则在编译的时候会提示链接错误。具体内容如下：

```
required_libraries = Analysis Core InstCombine IRReader Linker Object ProfileData Scalar Support TransformUtils Vectorize Obfuscation
```

修改Pass的CMakeLists.txt为静态库形式：

```
add_llvm_library(LLVMObfuscation
  SimplePass.cpp
  )

add_dependencies(LLVMObfuscation intrinsics_gen)
```

最后再编译一次。

那么我们可以这么去调用：

```
../build/bin/clang -mllvm -simplepass test.c -o after_test
```

基于Pass，我们可以做什么？ 我们可以编写自己的Pass去混淆代码，以增加他人反编译的难度。

<center><img src="http://7xtdl4.com1.z0.glb.clouddn.com/script_1482320959711.png"/></center>

我们可以把代码左上角的样子，变成右下角的样子，甚至更加复杂~ 

## 总结

上面说了那么说，来总结一下：

1.LLVM编译一个源文件的过程：

预处理 -> 词法分析 -> Token -> 语法分析 -> AST -> 代码生成 -> LLVM IR -> 优化 -> 生成汇编代码 -> Link -> 目标文件

2.基于LLVM，我们可以做什么？

1. 做语法树分析，实现语言转换OC转Swift、JS or 其它语言，字符串加密。
2. 编写ClangPlugin，命名规范，代码规范，扩展功能。
3. 编写Pass，代码混淆优化。

这篇只是一个简单的入门介绍，个人还需要深入去学习LLVM，再给大家分享，如有问题，欢迎拍砖~  

