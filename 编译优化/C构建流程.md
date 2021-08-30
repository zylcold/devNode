#build 

# 过程
预处理 - 词法分析 - 语法分析 - CodeGen - 生成汇编代码 - 汇编器 - 链接

# 预处理
预处理(preprocessor)

> 源代码 -> 源代码

预处理会替进行头文件引入，宏替换，注释处理，条件编译(＃ifdef)等操作。

```shell
xcrun clang -E main.c
```

# 词法分析
词法分析(lexical anaysis)

> 源代码(字符流) -> 词法单元(token)流

词法分析器读入源文件的字符流，将他们组织称有意义的词素(lexeme)序列，对于每个词素，此法分析器产生词法单元（token）作为输出。

```shell
$ xcrun clang -fmodules -fsyntax-only -Xclang -dump-tokens main.c
```


# 语法分析
语法分析(semantic analysis)

> 词法单元(token)流 -> 抽象语法树(AST)

词法分析的Token流会被解析成一颗抽象语法树(abstract syntax tree - AST)

有了抽象语法树，clang就可以对这个树进行分析，找出代码中的错误。比如类型不匹配，亦或Objective C中向target发送了一个未实现的消息。

```shell
$ xcrun clang -fsyntax-only -Xclang -ast-dump main.c | open -f
```

**AST是开发者编写clang插件主要交互的数据结构** ，clang也提供很多API去读取AST。更多细节： [Introduction to the Clang AST](https://clang.llvm.org/docs/IntroductionToTheClangAST.html) 。




# CodeGen

> 抽象语法树(AST) -> LR代码

CodeGen遍历语法树，生成LLVM LR代码。LLVM IR是前端的输出，后端的输入。


Objective C代码在这一步会进行runtime的桥接：property合成，ARC处理等。

LLVM会对生成的IR进行优化，优化会调用相应的Pass进行处理。Pass由多个节点组成，都是 [Pass](http://llvm.org/doxygen/classllvm_1_1Pass.html) 类的子类，每个节点负责做特定的优化，更多细节： [Writing an LLVM Pass](https://llvm.org/docs/WritingAnLLVMPass.html) 。


```shell
xcrun clang -S -emit-llvm main.c -o main.ll
```

# 生成汇编代码

> LR代码 -> 汇编代码(.s)

LLVM对LR进行优化后，会针对不同架构生成不同的目标代码，最后以汇编代码的格式输出

生成arm 64汇编：

```shell
$ xcrun clang -S main.c -o main.s
```

# 汇编器

> 汇编代码(.s) -> 机器代码(.o)

汇编器以汇编代码作为输入，将汇编代码转换为机器代码，最后输出目标文件(object file)。


```shell
$ xcrun clang -fmodules -c main.c -o main.o
```


# 链接

> 机器代码(.o) + dylib, a, tbd -> mach-o文件

连接器把编译产生的.o文件和（dylib,a,tbd）文件，生成一个mach-o文件

```shell
$ xcrun clang main.o -o main
```
