# LLVM是什么

LLVM项目是可重用(reusable)、模块化(modular)的编译器以及工具链技术(toolchain technologies)的集合。有人将其理解为“底层虚拟机(Low Level Virtual Machine)”的简称，但是[官方](https://llvm.org/)原话为：

> “The name “LLVM” itself is not an acronym; it is the full name of the project.”

意思是：LLVM不是首字母缩写，而是这整个项目的全名。

LLVM项目的发展起源于2000年伊利诺伊大学厄巴纳-香槟分校维克拉姆·艾夫（Vikram Adve）与克里斯·拉特纳（Chris Lattner）的研究，他们想要为所有静态及动态语言创造出动态的编译技术。2005年，苹果计算机雇用了克里斯·拉特纳及他的团队为苹果计算机开发应用程序系统，LLVM为现今Mac OS X及iOS开发工具的一部分。

# 1.LLVM&&clang安装

官网安装教程在[这里](http://clang.llvm.org/get_started.html)。这里简单介绍一下。

## 1.1Linux环境

### 1.1.1 安装前注意事项

由于本人使用的是虚拟机，所以在创建虚拟机的时候需要分配较大的内存和磁盘空间。这里踩了很多坑。

### 1.1.2 下载有关库

```
$ sudo apt-get install cmake
$ sudo apt-get install git
$ sudo apt-get install gcc
$ sudo apt-get install g++
```

### 1.1.3 下载项目源码

可以选择直接git整个工程，也可以去[官网](http://releases.llvm.org/download.html)下载源码然后自行安装。这里就按官网的git方法。

```
$ git clone https://github.com/llvm/llvm-project.git
```

下载好后会在当前目录下看到llvm-project文件夹。

[![LLVM](https://clheveningflow.github.io/2019/09/28/LLVM1/1.jpg)](https://clheveningflow.github.io/2019/09/28/LLVM1/1.jpg)

### 1.1.4 构建项目

```
$ cd llvm-project
```

创建构建目录

```
$ mkdir build
$ cd build
```

利用cmake构建

```
$ cmake -G <generator> [options] ../llvm
```

常用的generator有：

- Ninja ——生成Ninja 文件
- Unix Makefiles ——生成兼容make的makefile文件
- Visual Studio ——生成Visual Studio项目与解决方案
- Xcode ——用于生成Xcode项目

个人使用Unix Makefiles

常用的options有：

- DCMAKE_INSTALL_PREFIX=directory ——为目录指定要在其中安装LLVM工具和库的完整路径名（默认/usr/local）。
- DCMAKE_BUILD_TYPE=type ——type选项有Debug，Release，RelWithDebInfo和MinSizeRel。默认值为Debug。
- DLLVM_ENABLE_ASSERTIONS=On ——启用断言检查进行编译。
- DLLVM_ENABLE_PROJECTS=”…” ——要另外构建的LLVM子项目的列表，以’;’分隔。例如要构建LLVM，Clang，libcxx和libcxxabi，使用:`DLLVM_INSTALL_PROJECTS="clang;libcxx;libcxxabi"`
- DLLVM_TARGETS_TO_BUILD=”…” ——构建针对的平台的部分项目，以’;’分隔。默认面向所有平台编译(all)，指定只编译自己需要的CPU架构可以节省时间。

官方文档在[这](http://llvm.org/docs/CMake.html#options-and-variables)。

由于全部构建真的很耗费资源和时间，我使用的构建clang命令（可供参考）：

```
$ cmake -DCMAKE_BUILD_TYPE=Release -DLLVM_TARGETS_TO_BUILD="X86" -DLLVM_ENABLE_PROJECTS=clang -DLLVM_USE_LINKER=gold -G "Unix Makefiles" ../llvm
```

当然你如果愿意（~~而且设备跑得动~~）也可以：

```
$ cmake -G "Unix Makefiles" ../llvm
```

### 1.1.5 编译

```
$ make [-j <core>]
$ sudo make install
```

直接make也可以，但LLVM也支持并行编译，其中core取决于核心数。如：

```
$ make -j 4
```

这两步一般会很久……

### 1.1.6 测试

在编译结束后尝试在命令行中使用clang：

```
$ clang -v
```

本人得到结果如下：

```
clang version 10.0.0 
Target: x86_64-unknown-linux-gnu
Thread model: posix
InstalledDir: /usr/local/bin
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7.4.0
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/8
Selected GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7.4.0
Candidate multilib: .;@m64
Selected multilib: .;@m64
```

[![LLVM](https://clheveningflow.github.io/2019/09/28/LLVM1/2.jpg)](https://clheveningflow.github.io/2019/09/28/LLVM1/2.jpg)

编写一段C语言代码试试看(C++也可以):

```
//helloworld.c
#include <stdio.h>
int main() {
	printf("hello world\n");
	return 0;
}
```

用clang编译：

```
$ clang helloworld.c -o hello.out
$ ./hello.out
```

如果是C++代码则：

```
//helloworld.cpp
#include <iostream>
using namespace std;
int main() {
	cout << "hello world" << endl;
	return 0;
}
```

用clang编译（注意命令是clang++，本人刚开始只写clang提示编译错误…）：

```
$ clang++ helloworld.cpp -o hello.out
$ ./hello.out
```

[![LLVM](https://clheveningflow.github.io/2019/09/28/LLVM1/3.jpg)](https://clheveningflow.github.io/2019/09/28/LLVM1/3.jpg)

大功告成！

## 1.2.Windows环境

llvm+clang在windows下有两种类型，一种利用mingw编译，使用gcc的库和头文件；另一种利用vc编译，使用vc的库和头文件。推荐使用vc编译的版本，因为其程序更原生且不依赖于mingw，而只依赖于vc的dll库。

具体步骤[官方的教程](http://clang.llvm.org/get_started.html)已经说得很明白啦，下面简单整理介绍一下。

### 1.2.1 安装工具

- **Git.** 下载源代码。下载地址：https://git-scm.com/download
- **Cmake.** 用于生成Visual Studio的解决方案和项目文件。下载地址：https://cmake.org/download/
- **Visual Studio(推荐2017或之后版本).** 生成项目的容器。
- **Python.** 用于运行clang测试套件。下载地址：https://www.python.org/download/
- **GnuWin32 tools.** clang和LLVM的测试套件需要GNU工具。下载地址：http://getgnuwin32.sourceforge.net/

### 1.2.2 clone项目源代码

你也可以选择去github上直接download。

```
$ git clone https://github.com/llvm/llvm-progect.git
```

### 1.2.3 Cmake编译

在项目总目录下新建build目录（避免污染项目源代码）

```
$ mkdir build
$ cd build
```

运行Cmake。

```
$ cmake -DLLVM_ENABLE_PROJECTS=clang -G "Visual Studio 15 2017" -A x64 -Thost=x64 ..\llvm
```

- 如果使用Visual Studio 2019则改为 `-G "Visual Studio 16 2019"`。
- 如果需要适配x86则改为 `-A Win32`。

### 1.2.4 构建build

在Visual Studio中打开LLVM.sin，右击ALL_BUILD项目，选择生成，等待数小时。

### 1.2.5 设置环境变量

将llvm/debug/bin 添加到自己的环境变量中。

其余具体测试步骤请看[Hacking on clang - Testing using Visual Studio on Windows](http://clang.llvm.org/hacking.html#testingWindows)。

有待补充…

# 2.LLVM简介

用户文档：[llvm.org/docs/LangRef.html](https://llvm.org/docs/LangRef.html)

LLVM是基于静态单一分配的表示形式，可提供类型安全性、底层操作、灵活性，并且适配几乎所有高级语言，具有通用的代码表示。现在LLVM已经成为多个编译器和代码生成相关子项目的母项目。

> The LLVM code representation is designed to be used in three different forms: as an in-memory compiler IR, as an on-disk bitcode representation (suitable for fast loading by a Just-In-Time compiler), and as a human readable assembly language representation.

其中，LLVM提供了完整编译系统的中间层，并将中间语言（Intermediate Repressentation, IR）从编译器取出并进行最优化，最优化后的IR接着被转换及链接到目标平台的汇编语言。

我们知道，传统的编译器主要结构为：

[![LLVM](https://clheveningflow.github.io/2019/09/28/LLVM1/4.jpg)](https://clheveningflow.github.io/2019/09/28/LLVM1/4.jpg)

[传统编译器结构，图片摘自网络](https://clheveningflow.github.io/2019/09/28/LLVM1/4.jpg)



Frontend：前端，词法分析、语法分析、语义分析、生成中间代码

Optimizer：优化器，进行中间代码优化

Backend：后端，生成机器码

LLVM主要结构为：

[![LLVM](https://clheveningflow.github.io/2019/09/28/LLVM1/5.jpg)](https://clheveningflow.github.io/2019/09/28/LLVM1/5.jpg)

[LLVM结构，图片摘自网络](https://clheveningflow.github.io/2019/09/28/LLVM1/5.jpg)



也就是说，对于LLVM来说，不同的前后端使用统一的中间代码LLVM IR。如果需要支持一种新的编程语言/硬件设备，那么只需要实现一个新的前端/后端就可以了，而优化截断是一个通用的阶段，针对统一的LLVM IR，都不需要对于优化阶段修改。对比GCC，其前端和后端基本耦合在一起，所以GCC支持一门新的语言或者目标平台会变得很困难。

以下内容摘自维基百科：

> LLVM也可以在编译时期、链接时期，甚至是运行时期产生可重新定位的代码（Relocatable Code）。

> LLVM支持与语言无关的指令集架构及类型系统。每个在静态单赋值形式（SSA）的指令集代表着，每个变量（被称为具有类型的寄存器）仅被赋值一次，这简化了变量间相依性的分析。LLVM允许代码被静态的编译，包含在传统的GCC系统底下，或是类似JAVA等后期编译才将IF编译成机器代码所使用的即时编译（JIT）技术。它的类型系统包含基本类型（整数或是浮点数）及五个复合类型（指针、数组、向量、结构及函数），在LLVM具体语言的类型建制可以以结合基本类型来表示，举例来说，C++所使用的class可以被表示为结构、函数及函数指针的数组所组成。

> LLVM JIT编译器可以最优化在运行时期时程序所不需要的静态分支，这在一些部分求值（Partial Evaluation）的案例中相当有效，即当程序有许多选项，而在特定环境下其中多数可被判断为是不需要。这个特色被使用在Mac OS X Leopard（v10.5）底下OpenGL的管线，当硬件不支持某个功能时依然可以被成功地运作。OpenGL堆栈下的绘图程序被编译为IR，接着在机器上运行时被编译，当系统拥有高端GPU时，这段程序会进行极少的修改并将传递指令给GPU，当系统拥有低级的GPU时，LLVM将会编译更多的程序，使这段GPU无法运行的指令在本地端的中央处理器运行。LLVM增进了使用Intel GMA芯片等低端机器的性能。一个类似的系统发展于Gallium3D LLVMpipe，它已被合并到GNOME，使其可运行在没有GPU的环境。

> 根据2011年的一项测试，GCC在运行时期的性能平均比LLVM高10%。而2013年测试显示，LLVM可以编译出接近GCC相同性能的运行码。

# 3.Clang简介

Clang是LLVM针对C语言及其家族语言的前端(a C language family frontend for LLVM)。它的主要目标是提供一个GNU编译器套装（GCC）的替代品，支持GNU编译器大多数便已设置以及非官方语言拓展。项目包括Clang前端和Clang静态分析器。

> The Clang project provides a language front-end and tooling infrastructure for languages in the C language family (C, C++, Objective C/C++, OpenCL, CUDA, and RenderScript) for the LLVM project. Both a GCC-compatible compiler driver (clang) and an MSVC-compatible compiler driver (clang-cl.exe) are provided. You can get and build the source today.

Clang项目为LLVM项目中的C语言家族提供了一个语言前端和工具基础设施。其提供了兼容GCC和MSVC的编译器驱动程序（clang和clang-cl.exe）。
官方手册：http://clang.llvm.org/docs/UsersManual.html#basicusage

针对于GCC，Clang的优点有：

- 占用内存小
- 设计清晰简单，容易理解
- 编译速度快
- 设计偏向模块化，易于集成
- 诊断信息可读性强

## Clang(Clang++)使用

我们先随便写一段以下代码：

```
//test.cpp
#include <iostream>
#include <algorithm>

using namespace std;

int a[10] = {4,2,7,5,6,1,8,9,3,0};

int main() {
	for(int i = 0; i < 10; ++i)
		cout << a[i] << (i == 9?"\n":" ");
	sort(a,a+10);
	for(int i = 0; i < 10; ++i)
		cout << a[i] << (i == 9?"\n":" ");
	return 0;
}
```

[![LLVM](https://clheveningflow.github.io/2019/09/28/LLVM1/6.jpg)](https://clheveningflow.github.io/2019/09/28/LLVM1/6.jpg)

### 3.1.生成预处理文件：

```
$ clang++ -E test.cpp -o test.i
```

生成文件test.i（部分）如下：

[![LLVM](https://clheveningflow.github.io/2019/09/28/LLVM1/7.jpg)](https://clheveningflow.github.io/2019/09/28/LLVM1/7.jpg)

### 3.2.生成汇编程序：

```
$ clang++ -S test.i
```

[![LLVM](https://clheveningflow.github.io/2019/09/28/LLVM1/8.jpg)](https://clheveningflow.github.io/2019/09/28/LLVM1/8.jpg)

### 3.3.生成目标文件：

```
$ clang++ -c test.s
```

[![LLVM](https://clheveningflow.github.io/2019/09/28/LLVM1/9.jpg)](https://clheveningflow.github.io/2019/09/28/LLVM1/9.jpg)

### 3.4.生成可执行文件：

```
$ clang++ -o test.out test.o
```

[![LLVM](https://clheveningflow.github.io/2019/09/28/LLVM1/10.jpg)](https://clheveningflow.github.io/2019/09/28/LLVM1/10.jpg)

emm…~~和GCC大致还是一样的嘛~~ 非常的好用。

### 3.5.查看Clang编译的过程

```
$ clang -ccc-print-phases A.c
```

[![LLVM](https://clheveningflow.github.io/2019/09/28/LLVM1/11.jpg)](https://clheveningflow.github.io/2019/09/28/LLVM1/11.jpg)

- 0.获取输入：A.c文件，C语言
- 1.预处理器：处理define、include等
- 2.编译：生成中间代码（IR）
- 3.后端：生成汇编代码
- 4.汇编：生成目标代码
- 5.链接器：链接其他动态库

### 3.6.词法分析

```
$ clang -fmodules -E -Xclang -dump-tokens A.c
```

如图，写一个小函数对其进行词法分析。

[![LLVM](https://clheveningflow.github.io/2019/09/28/LLVM1/12.jpg)](https://clheveningflow.github.io/2019/09/28/LLVM1/12.jpg)

### 3.7.语法分析

```
$ clang -fmodules -fsyntax-only -Xclang -ast-dump A.c
```

生成语法树如下：

[![LLVM](https://clheveningflow.github.io/2019/09/28/LLVM1/13.jpg)](https://clheveningflow.github.io/2019/09/28/LLVM1/13.jpg)

有颜色区分还是比较美观的。

### 3.8.语义分析

生成LLVM IR。LLVM IR有3种表示形式（本质是等价的）

- (1).text:便于阅读的文本格式，类似于汇编语言，拓展名.ll
- (2).memory:内存格式
- (3).bitcode:二进制格式，拓展名.bc

生成text格式：

```
$ clang -S -emit-llvm A.c
```

[![LLVM](https://clheveningflow.github.io/2019/09/28/LLVM1/14.jpg)](https://clheveningflow.github.io/2019/09/28/LLVM1/14.jpg)

# 4.学习体会

安装LLVM绝对是一件痛苦的事情，我在Linux上安装LLVM+Clang起码花费有40小时（Windows上还没有成功55555）读了很久文档才搞清楚cmake的各种选项的用处，然后就是漫长的等待…

对编译原理的学习才刚刚起步，读了一下Clang的用户文档，很多选项都~~搞不明白，~~没有机会使用，但是经过使用后感觉LLVM真的是一个很强大的模块化编译器工具集合，而Clang的各种编译选项确实可以帮助理解编译的各个流程，光听编译原理课程是看不见一个实际的编译器是如何生成词法分析、语法分析的结果或者中间代码的。

LLVM不仅仅是编译器那么简单，我们可以利用其做各种NB的操作，比如开发新的编译器、新的编程语言、开发编译器插件、进行代码规范检查。它也绝对不是IOS领域特有的，因为它是一个高度模块化、可重用的组件，可适用于多门编程语言和多个硬件设备平台。可以说，大部分从事计算机工作的人都该懂点LLVM，而绝不仅仅只是开发者。

友情链接：

LLVM官网：[http://llvm.org](http://llvm.org/)

Clang官网：[http://clang.llvm.org](http://clang.llvm.org/)

LLVM用户文档：[llvm.org/docs/LangRef.html](https://llvm.org/docs/LangRef.html)

Clang用户文档：http://clang.llvm.org/docs/UsersManual.html