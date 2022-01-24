# 前言：学习 LLVM 有什么作用？

1.语法树（AST）分析

2.语言转换

- Clang 插件开发，代码检查（命名规范、代码规范）

- Pass 开发（代码优化、符号重命名、代码混淆）

3.创造一门新的编程语言

...

# 一、什么是 LLVM？

## 1.1 简介

**官方描述：**The LLVM Project is a collection of modular and reusable compiler and toolchain technologies. Despite its name, LLVM has little to do with traditional virtual machines. The name "LLVM" itself is not an acronym; it is the full name of the project.

1. LLVM 项目是模块化和可重用的编译器和工具链技术的集合
2. LLVM 与传统的虚拟机几乎没有关系。“LLVM” 这个名称本身并不是（Low Level Virtual Machine）首字母缩写词；它是项目的全名。



**历史：**LLVM 项目的发展起源于 2000 年，[伊利诺伊大学厄巴纳-香槟分校](https://zh.wikipedia.org/wiki/伊利诺伊大学厄巴纳-香槟分校)[维克拉姆·艾夫](https://zh.wikipedia.org/wiki/維克拉姆·艾夫)（Vikram Adve）与[克里斯·拉特纳](https://zh.wikipedia.org/wiki/克里斯·拉特納)（Chris Lattner）的研究，他们想要为所有静态及[动态语言](https://zh.wikipedia.org/wiki/动态语言)创造出动态的编译技术。LLVM是以 [BSD](https://zh.wikipedia.org/wiki/BSD授權)[许可](https://zh.wikipedia.org/wiki/BSD授權)来发展的开源软件。2005年，苹果雇佣了克里斯·拉特纳及他的团队为苹果电脑开发应用程序系统，LLVM 为现今 macOS 及 iOS 开发工具的一部分。



[克里斯·拉特纳](https://zh.wikipedia.org/wiki/克里斯·拉特納)（Chris Lattner），亦是 Swift 创造者

![img](https://cdn.nlark.com/yuque/0/2022/png/265739/1641282703412-43ca4e0d-886f-4330-b935-3359eda1d47d.png)

[克里斯·拉特纳](https://zh.wikipedia.org/wiki/克里斯·拉特納)硕士论文《LLVM：一个多阶段优化的编译器架构》（[LLVM](https://blog-1252789527.cos.ap-shanghai.myqcloud.com/article/Start From Scratch To Talk About LLVM/LLVM-2002-12-LattnerMSThesis.pdf)[: an infrastructure for multi-stage optimisation](https://blog-1252789527.cos.ap-shanghai.myqcloud.com/article/Start From Scratch To Talk About LLVM/LLVM-2002-12-LattnerMSThesis.pdf)）



**奖项：**LLVM 获得了 2012 年美国计算机协会（ACM）软件系统奖！该奖项由 ACM 每年颁发给全球一个软件系统。



**LLVM** **主要子项目：**

- **LLVM** **Core libraries：**LLVM 内核库提供了一套适合编译器系统的[中间语言](https://zh.wikipedia.org/wiki/中間語言)（Intermediate Representation，IR），在编译过程中有大量变换和优化都围绕其实现。经过变换和优化后的中间语言，可以转换为目标平台相关的[汇编语言](https://zh.wikipedia.org/wiki/汇编语言)代码。
- [Clang](http://clang.llvm.org/)**：**是一个基于 LLVM 架构的 C 语言家族（C / C ++ / Objective-C）编译器前端，其目地是提供一个快速编译的，非常有用的错误和警告消息，并为构建优秀的源代码级工具提供一个平台。

- [LLDB](http://lldb.llvm.org/)**：**是一个高性能的调试器。是 macOS 上 Xcode 中的默认调试器，并支持在台式机，iOS 设备和模拟器上调试 C，Objective-C 和 C ++。
- **libc++ 和 libc++** **ABI****：**项目提供了一个标准的符合性和高性能执行的 C++ 标准库。

- ....

除了LLVM的官方子项目外，还有许多其他项目，它们使用 LLVM 的组件来完成各种任务。

通过这些外部项目，可以使用 LLVM 来编译 Ruby，Python，Haskell，Rust，D，PHP，Pure，Lua和许多其他语言。

更多信息，请查看 [LLVM](https://llvm.org/) 官网

## 1.2 传统的编译器架构

![img](https://cdn.nlark.com/yuque/0/2022/png/265739/1641282703364-05357ca5-12ba-4367-8353-4ea2402c0df5.png)

**编译器的三段式设计：**

- **Frontend****（前端）**：主要工作是解析源代码：词法分析、语法分析、语义分析、检查错误，并构建抽象语法树（Abstract Syntax Tree AST）来表示输入代码。AST 可以选择性地转换为中间表达式，以便用于优化器。
- **Optimizer（优化器）**：负责优化代码，使代码更高效。例如消除冗余

- **Backend（后端）**：负责将优化器优化后的中间代码转化为目标机器指令集，最大化利用目标机器的特殊指令，提高代码的性能。例如：指令选择、寄存器分配和指令调度

三段式设计模式的优势体现在当编译器决定支持多种源语言或目标体系结构时。如果编译器在其优化器中使用公共代码表示，那么可以为任何可以编译到它的语言编写前端，也可以为任何可以从它编译的目标编写后端，如下图所示。

![img](https://cdn.nlark.com/yuque/0/2022/png/265739/1641282703382-ad3ff923-0b90-44e1-8c62-aa15857111be.png)

虽然这种三段式的编译器有很多优点，但是在实际中这一结构却从来没有被完美实现过。例如：Perl、Python、Ruby和Java的实现没有共享代码。此外，像Glasgow Haskell Compiler (GHC)和FreeBASIC这样的项目虽然可以重定向到多个不同的CPU，但是它们的实现非常依赖于某一种特定的编程语言（也就是它们所支持那种语言）。各种用于特殊用途的编译器技术被用于实现JIT编译器，用以实现图像处理、正则表达式、显卡驱动程序，或者被用于其他需要密集型CPU工作的领域。

**这个模型有三个主要的成功案例：**

1. **Java****虚拟机和 .NET 虚拟机：** 首先是Java虚拟机和.NET虚拟机。这两个系统都提供了JIT编译器、运行时支持和定义优良的字节码格式。这意味着任何可以编译成字节码格式的语言（有几十种格式）都可以利用优化器、JIT、运行时（runtime）。权衡的结果是，Java和.NET在实现**运行时（runtime）**的时候，几乎没有提供灵活性：它们都强制JIT编译、垃圾收集机制和使用非常特殊的对象模型。当编译与此模型不匹配的语言（如C语言、LLJVM项目）时，这会导致性能低下。
2. **翻译成C代码：** 第二个成功案例可能是最不幸的，但也是重用编译器技术最流行的方法：将输入源翻译成C代码（或其他语言），并用现有的C编译器进行处理。这允许重用优化器和代码生成器，提供良好的灵活性，控制了运行时，并且前端实现人员非常容易理解、实现和维护。但不幸的是，这样做会阻止异常处理的有效实现，使得调试的体验变得糟糕，降低编译速度，并且对于需要保证尾调用（或者C语言不支持的其他特性）的语言可能会有问题。

1. **GCC****编译器：** 此模型的最后一个成功实现是GCC。GCC支持许多前端和后端，并拥有一个活跃而广泛的开发者社区。GCC作为一个C编译器有着悠久的历史，它支持多个目标，并同时支持多语言实现。随着时间的推移，GCC社区正在向更纯粹而简洁的设计靠近。从GCC 4.4开始，它为优化器提供了一个新的表示（称为“GIMPLE元组”），与以前的表达形式相比，与前端的分离程度更高。此外，它的Fortran前端和Ada前端使用一中更简洁的AST。

这三种成功案例虽然看上去不错，但实质上它们还是存在着较大的局限性，因为它们被设计为单一而整体的程序。例如，我们不可能将GCC嵌入到其他应用程序中、将GCC用作运行时/JIT编译器，或者在不引入大部分编译器的情况下提取和重用GCC的代码片段。想要使用GCC的C++前端来生成文档、建立代码索引、重构代码或者制作静态分析工具，我们必须将GCC作为一个整体应用程序来使用。我们需要以XML的形式发送的信息，或者编写插件来将外部代码注入GCC进程中。

GCC不能作为可重用的库的原因有很多，包括大量使用全局变量、弱的常量、设计不佳的数据结构、不断扩展的代码库，以及使用宏来阻止将基本代码一次编译成支持多对前端/目标的能力。然而，最难解决的问题源自其早期设计时固有架构问题。具体来说，GCC存在分层问题和抽象泄漏的问题：后端遍历前端AST来生成调试信息，前端生成后端数据结构，而整个编译器又依赖于命令行接口设置的全局数据结构。

## 1.3 LLVM 架构

![img](https://cdn.nlark.com/yuque/0/2022/png/265739/1641282703384-26b4355f-758a-425e-aabf-c992da7a4b84.png)

**LLVM** **最初的优化的方向：**

| **存在的问题**                                               | **优化的方向**                                 |
| ------------------------------------------------------------ | ---------------------------------------------- |
| 中间表达形态（IR - Intermediate Representation）不能兼具通用性和强表达力 | 重新设计一种IR语言既包含低级信息也包含高级信息 |
| 整体化、高内聚的设计架构                                     | 改用模块化的设计模型                           |
| 难以用SDK和库的形式来重用                                    | 设计一种完美的调用机制                         |

**LLVM** **架构优点：**

- 不同的前端后端使用统一的中间代码 LLVM Intermediate Representation（LLVM IR）

1. 新增支持一种新的编程语言（前端语言），只需实现一个新的前端

2. 新增支持一种新的硬件设备（架构体系），只需要实现一个新的后端

1. 优化阶段是一个通用的阶段，针对的是不同的前端生成统一的中间代码

- 在某些平台中编译要比GCC快得多，例如，调试模式下编译Objective-C时，LLVM比GCC快3倍。

- 在生成抽象语法树（AST - Abstract Syntax Tree）时，占用的内存仅为GCC的1/5。

  - LLVM调试的调试信息表达更精准、更容易易分析和阅读。

  - LLVM被设计成一个模块化库，更容易嵌入IDE或进行重用。

  - LLVM很灵活，并且比GCC更容易扩展。

### 1.3.1 LLVM 的前端：Clang

**什么是** **Clang？**

- Clang 是指 LLVM 项目的编译器的前端部分，支持对 C 家族语言(C、C++、Objective-C)的编译。
- **Clang** **的功能包括：**词法分析、语法分析、语义分析、生成中间中间代码 LLVM Intermediate Representation (LLVM IR)。

**LLVM 目前已有众多的不同前端：**Clang、LLVM-GCC、GHC。

**相较** **GCC****，**Clang** **的优势：**

- **编译速度快：**clang 的编译速度较GCC快（Debug 模式下编译 OC 速度比 GCC 快3倍）
- **占用内存小：**clang 生成的 AST 所占用的内存是 GCC 的五分一左右

- **模块化设计：**clang 采用基于库的模块化设计，易于 IDE 集成以及模块重用和扩展
- **诊断信息可读性强：**clang 在编译过程中，创建并保留了详细的元数据，有利于调式、分析和阅读

模块化设计，各模块职责单一，功能清晰简单容易理解，可复用可移植性强，易于扩展增强

更多信息，请查看 [Clang](https://clang.llvm.org) 官网

### 1.3.2 LLVM 的中间代码：IR

LLVM 设计的最重要方面是 LLVM 中间代码（IR），它是用来在编译器中表示源码的一种形式。相当于是一种源到源的转换。下面是一个简单的 .ll 文件示例（[参考来源](http://www.aosabook.org/en/llvm.html)）：

一个简单的 C 函数：

```cpp
unsigned add(unsigned a, unsigned b) {
  return a + b;
}
```

与之对应的中间代码

```cpp
define i32 @add(i32 %a, i32 %b) {
entry:
  %tmp1 = add i32 %a, %b
  ret i32 %tmp1
}
```

### 1.3.3 LLVM 后端

![img](https://cdn.nlark.com/yuque/0/2022/png/265739/1641282703512-68e574d5-bb6b-4cef-985e-4f42bf8ea98c.png)

- 从传统编译器的三段式设计来看，后端应该是指：“生成目标代码”这一部分。

- - 但是很多书籍或者博客来把“优化器”和“生成目标代码” 统称为后端。《Gettiing Started with LLVM Core Libraries》。

- 在广义上 LLVM 指整个 LLVM 框架，狭义上 LLVM 指 LLVM 后端。

- - LLVM 项目是 围绕“LLVM Core libraries” 这个核心库而展开的，那时候还并没有前端，只有 LLVM 后端。从现在 LLVM 的项目来看，LLVM 指“LLVM 整个框架”这个定义是准确的！

# 二、LLVM 如何工作？（编译过程）

![img](https://cdn.nlark.com/yuque/0/2022/png/265739/1641282704540-b1d0cd51-01a6-45bf-9af0-cddb8956f2c2.png)

## 2.1 Objective-C 源文件编译过程

创建一个文件

```shell
$ vim main.m

#include <stdio.h>

#define NUMBER 3

int main() {
    // 这是一段注释
    int a = 1 + 2 + NUMBER;
    printf("NUMBER: %d\n", NUMBER);
    printf("a: %d", a);
    return 0;
}
```

查看编译过程：

```shell
$ clang -ccc-print-phases main.m

               +- 0: input, "main.m", objective-c
            +- 1: preprocessor, {0}, objective-c-cpp-output
         +- 2: compiler, {1}, ir
      +- 3: backend, {2}, assembler
   +- 4: assembler, {3}, object
+- 5: linker, {4}, image
6: bind-arch, "x86_64", {5}, image
```

\0. 输入源代码

1. **预处理器：**include、import、宏替换
2. **编译前端：**词法分析、语法分析、语义分析，编译成中间代码

1. **编译后端：**基于中间代码生成汇编
2. **汇编：**生成目标代码（机器码）

1. **链接：**链接其他库
2. **生成产物：**生成某个平台架构的产物

## 2.2 阶段一：编译前端（compiler）

**编译前端：**预处理、词法分析、语法分析、语义分析，编译成中间代码

### 2.2.1 预处理 (preprocessor)

**预处理器：**负责条件编译、源文件包含、宏替换、行控制、抛错、杂注和空指令; [详情](https://zhuanlan.zhihu.com/p/72515788)

```shell
$ clang -E main.m
```

预处理结果：

```shell
# 1 "main.m"
# 1 "<built-in>" 1
# 1 "<built-in>" 3
# 380 "<built-in>" 3
# 1 "<command line>" 1
# 1 "<built-in>" 2
# 1 "main.m" 2
//...
typedef signed char __int8_t;

typedef unsigned char __uint8_t;
typedef short __int16_t;
typedef unsigned short __uint16_t;
typedef int __int32_t;
typedef unsigned int __uint32_t;
typedef long long __int64_t;
typedef unsigned long long __uint64_t;

typedef long __darwin_intptr_t;
typedef unsigned int __darwin_natural_t;
# 70 "/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/usr/include/i386/_types.h" 3 4
typedef int __darwin_ct_rune_t;

typedef union {
 char __mbstate8[128];
 long long _mbstateL;
} __mbstate_t;

// ...

int main() {

    int a = 1 + 2 + 3;    
    printf("number = %d\n", 3);
    printf("a = %d", a);
    return 0;
}
```

### 2.2.2 词法分析 (Lexer)

**词法分析：**将预处理过的代码转化成一个个Token，比如左括号、右括号、等于、字符串等等。

```shell
$ clang -fmodules -fsyntax-only -Xclang -dump-tokens main.m
```

**词法分析结果：**

```shell
annot_module_include '#include <stdio.h>

'                Loc=<main.m:1:1>
int 'int'         [StartOfLine]        Loc=<main.m:5:1>
identifier 'main'         [LeadingSpace]        Loc=<main.m:5:5>
l_paren '('                Loc=<main.m:5:9>
r_paren ')'                Loc=<main.m:5:10>
l_brace '{'         [LeadingSpace]        Loc=<main.m:5:12>
int 'int'         [StartOfLine] [LeadingSpace]        Loc=<main.m:7:5>
identifier 'a'         [LeadingSpace]        Loc=<main.m:7:9>
equal '='         [LeadingSpace]        Loc=<main.m:7:11>
numeric_constant '1'         [LeadingSpace]        Loc=<main.m:7:13>
plus '+'         [LeadingSpace]        Loc=<main.m:7:15>
numeric_constant '2'         [LeadingSpace]        Loc=<main.m:7:17>
plus '+'         [LeadingSpace]        Loc=<main.m:7:19>
numeric_constant '3'         [LeadingSpace]        Loc=<main.m:7:21 <Spelling=main.m:3:16>>
semi ';'                Loc=<main.m:7:27>
identifier 'printf'         [StartOfLine] [LeadingSpace]        Loc=<main.m:8:5>
l_paren '('                Loc=<main.m:8:11>
string_literal '"NUMBER: %d\n"'                Loc=<main.m:8:12>
comma ','                Loc=<main.m:8:26>
numeric_constant '3'         [LeadingSpace]        Loc=<main.m:8:28 <Spelling=main.m:3:16>>
r_paren ')'                Loc=<main.m:8:34>
semi ';'                Loc=<main.m:8:35>
identifier 'printf'         [StartOfLine] [LeadingSpace]        Loc=<main.m:9:5>
l_paren '('                Loc=<main.m:9:11>
string_literal '"a: %d"'                Loc=<main.m:9:12>
comma ','                Loc=<main.m:9:19>
identifier 'a'         [LeadingSpace]        Loc=<main.m:9:21>
r_paren ')'                Loc=<main.m:9:22>
semi ';'                Loc=<main.m:9:23>
return 'return'         [StartOfLine] [LeadingSpace]        Loc=<main.m:10:5>
numeric_constant '0'         [LeadingSpace]        Loc=<main.m:10:12>
semi ';'                Loc=<main.m:10:13>
r_brace '}'         [StartOfLine]        Loc=<main.m:11:1>
eof ''                Loc=<main.m:11:2>
```

### 2.2.3 语法分析(Syntax) AST

**语法分析：**根据当前语言的语法，验证语法是否正确，并将所有节点组合成抽象语法树(AST)。

```shell
$ clang -fmodules -fsyntax-only -Xclang -ast-dump main.m
```

**语法分析结果：**

```shell
TranslationUnitDecl 0x7f8b67838008 <<invalid sloc>> <invalid sloc> <undeserialized declarations>
|-TypedefDecl 0x7f8b678388c0 <<invalid sloc>> <invalid sloc> implicit __int128_t '__int128'
| `-BuiltinType 0x7f8b678385a0 '__int128'
|-TypedefDecl 0x7f8b67838930 <<invalid sloc>> <invalid sloc> implicit __uint128_t 'unsigned __int128'
| `-BuiltinType 0x7f8b678385c0 'unsigned __int128'
|-TypedefDecl 0x7f8b678389d8 <<invalid sloc>> <invalid sloc> implicit SEL 'SEL *'
| `-PointerType 0x7f8b67838990 'SEL *'
|   `-BuiltinType 0x7f8b67838800 'SEL'
|-TypedefDecl 0x7f8b67838ab8 <<invalid sloc>> <invalid sloc> implicit id 'id'
| `-ObjCObjectPointerType 0x7f8b67838a60 'id'
|   `-ObjCObjectType 0x7f8b67838a30 'id'
|-TypedefDecl 0x7f8b67838b98 <<invalid sloc>> <invalid sloc> implicit Class 'Class'
| `-ObjCObjectPointerType 0x7f8b67838b40 'Class'
|   `-ObjCObjectType 0x7f8b67838b10 'Class'
|-ObjCInterfaceDecl 0x7f8b67838bf0 <<invalid sloc>> <invalid sloc> implicit Protocol
|-TypedefDecl 0x7f8b67838f90 <<invalid sloc>> <invalid sloc> implicit __NSConstantString 'struct __NSConstantString_tag'
| `-RecordType 0x7f8b67838d60 'struct __NSConstantString_tag'
|   `-Record 0x7f8b67838cc0 '__NSConstantString_tag'
|-TypedefDecl 0x7f8b69033c48 <<invalid sloc>> <invalid sloc> implicit __builtin_ms_va_list 'char *'
| `-PointerType 0x7f8b69033c00 'char *'
|   `-BuiltinType 0x7f8b678380a0 'char'
|-TypedefDecl 0x7f8b69033f58 <<invalid sloc>> <invalid sloc> implicit __builtin_va_list 'struct __va_list_tag [1]'
| `-ConstantArrayType 0x7f8b69033f00 'struct __va_list_tag [1]' 1
|   `-RecordType 0x7f8b69033d40 'struct __va_list_tag'
|     `-Record 0x7f8b69033ca0 '__va_list_tag'
|-ImportDecl 0x7f8b69034780 <main.m:1:1> col:1 implicit Darwin.C.stdio
`-FunctionDecl 0x7f8b69034810 <line:5:1, line:11:1> line:5:5 main 'int ()'
  `-CompoundStmt 0x7f8b691aefc8 <col:12, line:11:1>
    |-DeclStmt 0x7f8b691aecb0 <line:7:5, col:27>
    | `-VarDecl 0x7f8b69034928 <col:5, line:3:16> line:7:9 used a 'int' cinit
    |   `-BinaryOperator 0x7f8b69034a10 <col:13, line:3:16> 'int' '+'
    |     |-BinaryOperator 0x7f8b690349d0 <line:7:13, col:17> 'int' '+'
    |     | |-IntegerLiteral 0x7f8b69034990 <col:13> 'int' 1
    |     | `-IntegerLiteral 0x7f8b690349b0 <col:17> 'int' 2
    |     `-IntegerLiteral 0x7f8b690349f0 <line:3:16> 'int' 3
    |-CallExpr 0x7f8b691aedd8 <line:8:5, col:34> 'int'
    | |-ImplicitCastExpr 0x7f8b691aedc0 <col:5> 'int (*)(const char *, ...)' <FunctionToPointerDecay>
    | | `-DeclRefExpr 0x7f8b691aecc8 <col:5> 'int (const char *, ...)' Function 0x7f8b69034a38 'printf' 'int (const char *, ...)'
    | |-ImplicitCastExpr 0x7f8b691aee20 <col:12> 'const char *' <NoOp>
    | | `-ImplicitCastExpr 0x7f8b691aee08 <col:12> 'char *' <ArrayToPointerDecay>
    | |   `-StringLiteral 0x7f8b691aed28 <col:12> 'char [12]' lvalue "NUMBER: %d\n"
    | `-IntegerLiteral 0x7f8b691aed50 <line:3:16> 'int' 3
    |-CallExpr 0x7f8b691aef20 <line:9:5, col:22> 'int'
    | |-ImplicitCastExpr 0x7f8b691aef08 <col:5> 'int (*)(const char *, ...)' <FunctionToPointerDecay>
    | | `-DeclRefExpr 0x7f8b691aee38 <col:5> 'int (const char *, ...)' Function 0x7f8b69034a38 'printf' 'int (const char *, ...)'
    | |-ImplicitCastExpr 0x7f8b691aef68 <col:12> 'const char *' <NoOp>
    | | `-ImplicitCastExpr 0x7f8b691aef50 <col:12> 'char *' <ArrayToPointerDecay>
    | |   `-StringLiteral 0x7f8b691aee98 <col:12> 'char [6]' lvalue "a: %d"
    | `-ImplicitCastExpr 0x7f8b691aef80 <col:21> 'int' <LValueToRValue>
    |   `-DeclRefExpr 0x7f8b691aeeb8 <col:21> 'int' lvalue Var 0x7f8b69034928 'a' 'int'
    `-ReturnStmt 0x7f8b691aefb8 <line:10:5, col:12>
      `-IntegerLiteral 0x7f8b691aef98 <col:12> 'int' 0
```

### 2.2.4 生成中间代码(IR)

CodeGen 负责将语法树从顶至下遍历，翻译成中间代码IR，IR是LLVM Frontend的输出，也是LLVM Backerend的输入，桥接前后端。

```shell
$ clang -S -fobjc-arc -emit-llvm main.m -o main.ll
```

**中间代码（IR）：**

```shell
[1] 15386
; ModuleID = 'main.m'
source_filename = "main.m"
target datalayout = "e-m:o-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-apple-macosx11.3.0"

@.str = private unnamed_addr constant [12 x i8] c"number = %d\00", align 1
@.str.1 = private unnamed_addr constant [7 x i8] c"a = %d\00", align 1

; Function Attrs: noinline optnone ssp uwtable
define i32 @main() #0 {
  %1 = alloca i32, align 4
  %2 = alloca i32, align 4
  store i32 0, i32* %1, align 4
  store i32 6, i32* %2, align 4
  %3 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([12 x i8], [12 x i8]* @.str, i64 0, i64 0), i32 3)
  %4 = load i32, i32* %2, align 4
  %5 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([7 x i8], [7 x i8]* @.str.1, i64 0, i64 0), i32 %4)
  ret i32 0
}

declare i32 @printf(i8*, ...) #1

attributes #0 = { noinline optnone ssp uwtable "darwin-stkchk-strong-link" "disable-tail-calls"="false" "frame-pointer"="all" "less-precise-fpmad"="false" "min-legal-vector-width"="0" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="true" "probe-stack"="___chkstk_darwin" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+cx8,+fxsr,+mmx,+sahf,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "tune-cpu"="generic" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #1 = { "darwin-stkchk-strong-link" "disable-tail-calls"="false" "frame-pointer"="all" "less-precise-fpmad"="false" "no-infs-fp-math"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="true" "probe-stack"="___chkstk_darwin" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+cx8,+fxsr,+mmx,+sahf,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "tune-cpu"="generic" "unsafe-fp-math"="false" "use-soft-float"="false" }

!llvm.module.flags = !{!0, !1, !2, !3, !4, !5, !6, !7}
!llvm.ident = !{!8}

!0 = !{i32 2, !"SDK Version", [2 x i32] [i32 11, i32 3]}
!1 = !{i32 1, !"Objective-C Version", i32 2}
!2 = !{i32 1, !"Objective-C Image Info Version", i32 0}
!3 = !{i32 1, !"Objective-C Image Info Section", !"__DATA,__objc_imageinfo,regular,no_dead_strip"}
!4 = !{i32 1, !"Objective-C Garbage Collection", i8 0}
!5 = !{i32 1, !"Objective-C Class Properties", i32 64}
!6 = !{i32 1, !"wchar_size", i32 4}
!7 = !{i32 7, !"PIC Level", i32 2}
!8 = !{!"Apple clang version 13.0.0 (clang-1300.0.29.3)"}
```

## 2.3 阶段二：优化器

**优化器：**负责中间代码优化

### 2.3.1 代码优化(Opt)

例如 Xcode 中开启了bitcode，那么苹果后台拿到的就是这种中间代码，苹果可以对 bitcode 做进一步的优化。

```shell
$ clang -emit-llvm -c main.m -o main.bc
```

**生成** **bc****：**

[file] main.bc

## 2.4 阶段三：编译后端（backend）

**编译后端：**负责中间代码优化、生成目标文件

### 2.4.1 汇编 代码生成器（assembler）

```shell
// 生成汇编代码
$ clang -S -fobjc-arc main.m -o main.s 
// 生成目标文件
$ clang -fmodules -c main.m -o main.o
```

**汇编结果：**

```shell
        .section        __TEXT,__text,regular,pure_instructions
        .build_version macos, 11, 3        sdk_version 11, 3
        .globl        _main                           ## -- Begin function main
        .p2align        4, 0x90
_main:                                  ## @main
        .cfi_startproc
## %bb.0:
        pushq        %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset %rbp, -16
        movq        %rsp, %rbp
        .cfi_def_cfa_register %rbp
        subq        $16, %rsp
        movl        $0, -4(%rbp)
        movl        $6, -8(%rbp)
        leaq        L_.str(%rip), %rdi
        movl        $3, %esi
        movb        $0, %al
        callq        _printf
        movl        -8(%rbp), %esi
        leaq        L_.str.1(%rip), %rdi
        movb        $0, %al
        callq        _printf
        xorl        %eax, %eax
        addq        $16, %rsp
        popq        %rbp
        retq
        .cfi_endproc
                                        ## -- End function
        .section        __TEXT,__cstring,cstring_literals
L_.str:                                 ## @.str
        .asciz        "number = %d\n"

L_.str.1:                               ## @.str.1
        .asciz        "a = %d"

        .section        __DATA,__objc_imageinfo,regular,no_dead_strip
L_OBJC_IMAGE_INFO:
        .long        0
        .long        64

.subsections_via_symbols
```

### 2.4.2 链接成可执行文件（linker）

```shell
$ clang main.o -o main
```

# 三、LLVM 环境搭建（源码编译）

## 3.1 Step1：下载 LLVM 源码

```shell
$ git clone https://github.com/llvm/llvm-project.git
$ git checkout release/13.x
```

## 3.2 Step2：准备编译环境

安装 brew

```shell
$ /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

查看 brew 版本，输出版本信息，则代表 brew 安装成功

```shell
$ llvm-xcode brew --version
Homebrew 3.3.2
Homebrew/homebrew-core (git revision 1db152a908e; last commit 2021-11-04)
Homebrew/homebrew-cask (git revision 02f34e0c85; last commit 2021-11-04)
```

关于 HomeBrew 更多信息，请查看： [HomeBrew 官网](https://brew.sh/)

安装 CMake

```shell
$ brew cmake
```

查看 CMake 版本，输出版本信息，则代表 CMake 安装成功

```shell
$ cmake --version
cmake version 3.21.4
ke suite maintained and supported by Kitware (kitware.com/cmake).
```

关于 CMake 更多信息，请查看： [CMake 官网](https://cmake.org/)

安装 Ninja，若采用方案一编译 LLVM 可以不用安装 Ninja，建议采用方案二：

```shell
$ brew ninja
```

查看 Ninja 版本，输出版本信息，则代表 ninja 安装成功

```shell
$ ninja --version 1.10.2
```

关于 Ninja 更多信息，请查看：[Ninja 官网](https://ninja-build.org/)

## 3.2 Step3：编译 LLVM

推荐使用 方案二： Ninja 编译 LLVM

#### 方案一：使用 CMake & Make build LLVM (1h)

```shell
// TODO: CMake build...
```

#### 方案二：Ninja build LLVM (30min)

在 LLVM 源码同级目录下新建一个llvm-build目录，用来生成 build.ninja

```shell
$ mkdir llvm-build
```

在 LLVM 源码同级目录下新建一个llvm-release目录，用来生成安装 LLVM

```shell
$ mkdir llvm-release
```

进入llvm-build目录，生成 Ninja 模板

```shell
$ cd llvm-build
$ cmake -G Ninja -DLLVM_ENABLE_PROJECTS='clang' -DCMAKE_INSTALL_PREFIX=../llvm-release ../llvm-project/llvm
```

- 更多的配置请参考：[Getting Started with the](https://llvm.org/docs/GettingStarted.html#getting-the-source-code-and-building-llvm)[LLVM](https://llvm.org/docs/GettingStarted.html#getting-the-source-code-and-building-llvm)[System](https://llvm.org/docs/GettingStarted.html#getting-the-source-code-and-building-llvm)
- CMake 详细配置请参考：[Building](https://llvm.org/docs/CMake.html)[LLVM](https://llvm.org/docs/CMake.html)[with CMake](https://llvm.org/docs/CMake.html)

Ninja 编译 LLVM

```shell
$ ninja
```

编译完成后，llvm-build目录大概 39 G。请保证有足够的存储空间

Ninjia install

```shell
$ ninja install
```

安装完成后，llvm-release目录大概 24 G。请保证有足够的存储空间

#### 方案三：Xcode 编译（1h）

在 LLVM 源码同级目录下新建一个llvm-xcode目录，用来生成 Xcode 项目

```shell
$ mkdir llvm-xcode
```

进入llvm-xcode目录，生成 Xcode 项目

```shell
$ cd llvm-xcode 
$ cmake -G Xcode -DLLVM_ENABLE_PROJECTS='clang' ../llvm-project/llvm
```

首次打开 LLVM.xcodeproj 项目，Xcode 会弹出窗口提示对 Scheme 进行配置。选择自动创建 Schemes 即可。

![img](https://cdn.nlark.com/yuque/0/2022/png/265739/1641282704560-a9ca9d32-3572-4482-aec7-8cc59454887a.png)![img](https://cdn.nlark.com/yuque/0/2022/png/265739/1641282704578-70f412ea-79af-401a-801f-59b03ff8bca1.png)

打开LLVM.xcodeproj 工程，LLVM项目的目录结构如下

![img](https://cdn.nlark.com/yuque/0/2022/png/265739/1641282704803-793d4c83-bcb1-4112-a95c-bf00a662c48f.png)

Clang libraries 目录下分布了大量的模块，这些模块都可以单独生成一个库文件

例如：clangBasic、clangLex、clangAST、clangSema等等。

Clang 所具备的功能例如：参数解析、词法分析、语法分析、生成语法树、源文件读写等对应的源码都在 Clang libraries 模块下

![img](https://cdn.nlark.com/yuque/0/2022/png/265739/1641282704820-c53e46ee-92bc-4f24-98eb-91893e17a9da.png)

最后，选择Scheme ALL_BUILD，按下Command+B开始构建。

![img](https://cdn.nlark.com/yuque/0/2022/png/265739/1641282705015-2c936f85-b8f7-4370-a7dd-e2540a881d5c.png)

接下来，可以选择泡杯咖啡休息一会了。Xcode 编译 LLVM 可能会持续一个小时左右。

# 四、LLVM 应用场景

**语法树分析、语言转换：**libclang、libTooling。

1. 1. 参考 http://clang.llvm.org/docs/Tooling.html

**Clang** **插件开发**

1. 1. 可以用来做代码检查（命名规范、代码规范），安装包瘦身等等各种事情。 编写插件可以参考官方教程
   2. 参考： http://clang.llvm.org/docs/ClangPlugins.htmlhttp://clang.llvm.org/docs/ExternalClangExamples.htmlhttp://clang.llvm.org/docs/RAVFrontendAction.html

**Pass 开发：**可以用来做代码优化、代码混淆等。

1. 1. 官方 Pass 教程 https://llvm.org/docs/WritingAnLLVMPass.html

**开发新的编程语言**

1. [用](https://llvm-tutorial-cn.readthedocs.io/en/latest/index.html)[LLVM](https://llvm-tutorial-cn.readthedocs.io/en/latest/index.html)[开发新语言](https://llvm-tutorial-cn.readthedocs.io/en/latest/index.html) 原文：http://llvm.org/docs/tutorial/index.html 译文：https://llvm-tutorial-cn.readthedocs.io/en/latest/index.html

- [DynamicCocoa：滴滴 iOS 动态化方案的诞生与起航](http://www.cocoachina.com/articles/18400)
- [腾讯 OCS：史上最疯狂的 iOS 动态化方案](https://its201.com/article/guojin08/54310858)
- **蚂蚁金服：**进化的覆盖率---代码实时染色系统 [PPT](https://spider.doc88.com/p-14661890470922.html)[MTSC2019](https://www.itdks.com/Home/Act/apply?id=3080)[大会视频](https://www.itdks.com/Course/detail?id=117033)
- [打造基于](https://www.jianshu.com/p/01c988cae897)[Clang](https://www.jianshu.com/p/01c988cae897)[LibTooling 的 iOS 自动打点系统 CLAS](https://www.jianshu.com/p/01c988cae897)

# 五、LLVM 实战：ObjC 代码混淆 | 添加命名空间（clang-obfuscator）

## 5.1 Clang AST 初识

#### 5.1.1 clang

Clang 项目非常庞大。仅仅是 Clang AST 相关代码就超过 10W+ 行代码。如何利用 Clang 实现 AST 分析工作，这里可以参考官网提供的文档 [Choosing the Right Interface for Your Application](https://clang.llvm.org/docs/Tooling.html) ，以下是三种方式：

1. **LibClang**

1. 1. 提供 C 语言的稳定接口，支持Python Binding。AST 并不完整，不能完全掌控 Clang AST。

1. **Clang Plugins**

1. 1. 提供 C++ 接口，更新快，不能保留上下文信息。插件的存在形式是一个动态链接库，不能在构建环境外独立存在。

1. **LibTooling**

1. 1. 提供 C++ 接口，更新快，可以通过标准的 main() 函数作为入口，可独立运行，能够完全掌控 AST，相比 Plugin 更容易设置。

这里我们选择可独立运行并且能完全掌控 AST 的 LibTooling 作为 clang-obfuscator 的基础。

#### 5.1.2 AST

在使用 Clang 的学习过程中，基本的概念便是表示 AST 的节点类型，这里重要的几点是：

1. **ASTContext。**

1. 1. ASTContext 是编译实例用来保存 AST 相关信息的一种结构，也包含了编译期间的符号表。我们可以通过 TranslationUnitDecl * getTranslationUnitDecl()： 方法得到整个翻译单元的 AST 的入口节点。

1. **节点类型。**

1. 1. AST 通过三组核心类构建：Decl (declarations)、Stmt (statements)、Type (types)。其它节点类型并不会从公共基类继承，因此，没有用于访问树中所有节点的通用接口。

1. **遍历方式。**

1. 1. 为了分析 AST，我们需要遍历语法树。Clang 提供了两种方式：RecursiveASTVisitor 和 ASTMatcher。RecursiveASTVisitor 能够让我们以深度优先的方式遍历 Clang AST 节点。我们可以通过扩展类并实现所需的 VisitXXX 方法来访问特定节点。

ASTMatcher API 提供了一种域特定语言（DSL）来构建基于 Clang AST 的谓词，它能高效地匹配到我们感兴趣的节点。

除了这两种方式外，LibClang 也提供了 Cursors 来遍历 AST。更多细节内容可以前往 ：[clang.llvm.org](http://clang.llvm.org/)。

## 5.2 clang-obfuscator 开发

### 5.2.1 创建 clang-obfuscator 项目

进入 llvm-project/clang/tools

```shell
$ cd llvm-project/clang/tools
```

创建 clang-obfuscator 目录

```shell
$ mkdir clang-obfuscator
```

进入 clang-obfuscator 目录，创建 obfuscator.h & obfuscator.cpp 文件

```shell
$ cd clang-obfuscator 
# 创建混淆文件 
$ touch Obfuscator.h && touch Obfuscator.cpp
```

### 5.2.2 创建 CMakeList.txt 配置文件

创建 CMakeList.txt 配置文件，并写入下面内容

```shell
$ vim CMakeLists.txt 

set(LLVM_LINK_COMPONENTS
  Support
)

# 对应的 cpp 文件都要加进来
add_clang_executable(ClangObfuscator
  Obfuscator.cpp 
)

# 将目标文件与库文件进行链接
target_link_libraries(ClangObfuscator
    PRIVATE
    clangAST
    clangBasic
    clangDriver
    clangFormat
    clangLex
    clangParse
    clangSema
    clangFrontend
    clangTooling
    clangToolingCore
    clangRewrite
    clangRewriteFrontend
)

if(UNIX)
  set(CLANGXX_LINK_OR_COPY create_symlink)
else()
  set(CLANGXX_LINK_OR_COPY copy)
endif()
```

代码中出现的指令：

- set：用来显式的定义变量，用变量代替值，例子中定义LLVM_LINK_COMPONENTS代替后面的字符串。
- add_clang_executable：将名为 Obfuscator.cpp 的源文件编译成一个名称为 ClangObfuscator 的可执行文件。

- target_link_libraries：将目标文件与库文件进行连接。

### 5.2.3 配置子模块

修改上层目录的 CMakeLists.txt

```shell
$ cd .. 
$ echo 'add_subdirectory(clang-obfuscator)' >> ./CMakeLists.txt
```

查看CMakeLists.txt文件如下：

```shell
$ cat CMakeLists.txt
create_subdirectory_options(CLANG TOOL)

add_clang_subdirectory(diagtool)
// ...
// ...
# We support checking out the clang-tools-extra repository into the 'extra'
# subdirectory. It contains tools developed as part of the Clang/LLVM project
# on top of the Clang tooling platform. We keep them in a separate repository
# to keep the primary Clang repository small and focused.
# It also may be included by LLVM_EXTERNAL_CLANG_TOOLS_EXTRA_SOURCE_DIR.
add_llvm_external_project(clang-tools-extra extra)

# libclang may require clang-tidy in clang-tools-extra.
add_clang_subdirectory(libclang)

add_clang_subdirectory(amdgpu-arch)
add_subdirectory(clang-obfuscator)
```

### 5.2.4 重新生成LLVM.xcodeproj

进入 llvm-xcode 目录 

```shell
$ cd ../../../llvm-xcode
```

重新生成 LLVM.xcodeproj

```shell
$ cmake -G Xcode -DLLVM_ENABLE_PROJECTS='clang' ../llvm-project/llvm
```

LLVM.xcodeproj重新生成后，会多出一个ClangObfuscator的Target并且Clang executables目录中多出如下结构:

![img](https://cdn.nlark.com/yuque/0/2022/png/265739/1641282705595-ee2cba65-beb0-4496-92f6-b7b92b5a82bd.png)

接下来选中ClangObfuscator就可以开始编码了：

![img](https://cdn.nlark.com/yuque/0/2022/png/265739/1641282705586-868448c6-02eb-4c7d-994b-0f89c6cd390e.png)



# 六、规划

// TODO: 

# 七、参考

- **LLVM**

- - [The](https://llvm.org/)[LLVM](https://llvm.org/)[Compiler Infrastructure](https://llvm.org/)
  - [The Architecture of Open Source Applications /](http://www.aosabook.org/en/llvm.html)[LLVM](http://www.aosabook.org/en/llvm.html)

- - [The LLDB Debugger](https://lldb.llvm.org/)
  - [Clang: a C language family frontend for](https://clang.llvm.org/)[LLVM](https://clang.llvm.org/)

- **Clang**

- - [Clang 11 documentation](http://clang.llvm.org/docs/index.html#)
  - [Clang 11 documentation-Choosing the Right Interface for Your Application](http://clang.llvm.org/docs/Tooling.html)

- - [How to write RecursiveASTVisitor based ASTFrontendActions.](http://clang.llvm.org/docs/RAVFrontendAction.html)

- **AST**

- - [Introduction to the Clang](http://clang.llvm.org/docs/IntroductionToTheClangAST.html)[AST](http://clang.llvm.org/docs/IntroductionToTheClangAST.html)
  - [How to write RecursiveASTVisitor based ASTFrontendActions.](http://clang.llvm.org/docs/RAVFrontendAction.html)

- - [Tutorial for building tools using LibTooling and LibASTMatchers](http://clang.llvm.org/docs/LibASTMatchersTutorial.html)
  - [Matching the Clang](http://clang.llvm.org/docs/LibASTMatchers.html)[AST](http://clang.llvm.org/docs/LibASTMatchers.html)

- - [ASTImporter: Merging Clang ASTs](http://clang.llvm.org/docs/LibASTImporter.html)
  - [AST](https://clang.llvm.org/docs/LibASTMatchersReference.html#narrowing-matchers)[Matcher Reference](https://clang.llvm.org/docs/LibASTMatchersReference.html#narrowing-matchers)

- **LLVM** **环境搭建**

- - [LLVM](https://github.com/llvm/llvm-project)[-Project](https://github.com/llvm/llvm-project)
  - [Getting Started with the](https://llvm.org/docs/GettingStarted.html#getting-the-source-code-and-building-llvm)[LLVM](https://llvm.org/docs/GettingStarted.html#getting-the-source-code-and-building-llvm)[System](https://llvm.org/docs/GettingStarted.html#getting-the-source-code-and-building-llvm)

- - [CMake Tutorial](https://cmake.org/cmake/help/latest/guide/tutorial/index.html#id1)
  - [CMake](https://www.hahack.com/codes/cmake/)[入门实战](https://www.hahack.com/codes/cmake/)
