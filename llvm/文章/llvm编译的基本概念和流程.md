近期跟着《LLVM Cookbook》学习了一下LLVM相关的内容，趁着学完还算熟悉，赶紧写一下笔记和总结，方便以后的回顾。LLVM的全称是Low Level Virtual Machine，名字已经解释了好多内容。作为一个编译器的基础框架，提供了各种工具和编译的基础设施，允许我们自定义前端，实现特定的优化方法，并绑定自己的后端。特别是MLIR和IREE的发布，加上当下芯片行业越来越热，LLVM在社区和产业界的影响力也随之扩大，了解和学习LLVM还是很有必要的。

## 代码形式及转换关系

LLVM涉及到的内容比较多，这里把东西先列出来，

### 代码形式的基本概念

1. **main.c** c源代码，对于llvm来说即前端代码，能够在clang或其他前端的处理下生成LLVM的IR和bitcode，从而进行后续的操作。
2. **main.ll** llvm IR, 可读的llvm汇编代码。
3. **main.bc** llvm bitcode，LLVM的主要表示，后续工具能够直接在这一层表示上运行，又被称为llvm字节码，是LLVM IR编码成位流的编码格式。
4. **main.s** 目标架构下的汇编代码，不同架构，不同格式下均不一致，这也是llvm想为用户屏蔽的底层信息。x86和arm等均有自己的表示。
5. **main.o** 目标架构下的object文件，经过编译生成的二进制文件，能够进行链接操作。
6. **a.out** 对目标架构下的object文件链接后的产物，能够在目标架构下执行。

用图示表示的会更清晰一些，这里画了一下几种代码格式之间的转换关系

[![img](http://qn.throneclay.top/image/jpg/llvm_file.jpg_out.jpg)](http://qn.throneclay.top/image/jpg/llvm_file.jpg_out.jpg)

### 用到的常用工具

llvm的文档工作还是做的非常好的，这里节选了llvm官方文档对自家工具的介绍，重点说一下常用的一些命令，详细全文见https://llvm.org/docs/CommandGuide/index.html

1. **clang/clang++** c语言的前端工具，能够完成c/c++代码到LLVM的转换。输入是c/c++的源码。

   ```
   clang --emit-llvm -S -c main.c -o main.ll
   clang --emit-llvm -c main.c -o main.bc
   ```

2. **opt** llvm 优化器，能够对Bitcode执行优化(PASS)的工具。输入是LLVM bitcode，输出是优化后的LLVM bitcode

3. **llvm-as** llvm汇编器，输入是LLVM IR，输出为LLVM bitcode。

4. **llvm-dis** llvm反汇编器，输入是bitcode，输出为LLVM IR。

5. **llc** llvm静态编译器，根据特定的后端，将bitcode编译对应的汇编代码

6. **lli** bitcode立即执行工具，使用jit或解释器执行一个bitcode。输入为bitcode，输出执行效果。

7. **llvm-link** llvm链接器，将多个bitcode链接为一个bitcode，输入为多个bitcoede，输出一个链接后的bitcode。

8. **llvm-config** 输出llvm编译选项，根据选项，输出合适的llvm的编译选项。

[![img](http://qn.throneclay.top/image/jpg/two_links_out.jpg)](http://qn.throneclay.top/image/jpg/two_links_out.jpg)

### 编译链接时会用到的LLVM lib

通常这部分会使用llvm-config来代替调，但了解这些库的功能对于未来可能的debug还是很有帮助的。（参考 http://faculty.sist.shanghaitech.edu.cn/faculty/songfu/course/spring2018/CS131/llvm.pdf -Getting to know LLVM’s basic libraries）

1. **libLLVMCore** 包含LLVM IR(bitcode)相关的逻辑，IR构造（data layout，instructions，basic block和functions）以及IR校验器。
2. **libLLVMAnalysis** 包含IR的分析过程，如别名分析，依赖分析，常量折叠，循环信息，内存依赖分析和指令简化。
3. **libLLVMCodeGen** 该库实现与目标无关的代码生成和机器级别的分析和转换。
4. **libLLVMTarget** 对目标机器信息提供通用的抽象封装接口。实现是在libLLVMCodeGen中的通用后端算法和libLLVM[target Marchine]CodeGen库中的特定后端算法。
5. **libLLVMX86CodeGen** 如上面所说，这个库里是x86目标的特定后端算法，包括代码生成信息、转换和分析过程。
6. **libLLVMARMCodeGen** 这个库里是Arm目标的特定后端算法，包括代码生成信息、转换和分析过程。这种库还有很多，就不分别列举了。
7. **libLLVMSupport** 库里包含通用工具集合。包括错误，整型和浮点处理、命令行解析、调试、文件支持和字符串处理。

## pipeline

完整的llvm工作pipeline如下：

[![img](http://qn.throneclay.top/image/jpg/llvm-pipeline_out.jpg)](http://qn.throneclay.top/image/jpg/llvm-pipeline_out.jpg)
