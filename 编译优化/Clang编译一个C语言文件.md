#build 

## 过程
预处理 - 词法分析 - 语法分析 - CodeGen - 生成汇编代码 - 汇编器 - 链接

## 预处理
预处理(preprocessor)
> 源代码 -> 源代码

预处理会替进行**头文件引入**，**宏替换**，**注释处理**，**条件编译(＃ifdef)**等操作。

```shell
xcrun clang -E <inputName>.c
```

# 词法分析
词法分析(lexical anaysis)
> 源代码(字符流) -> 词法单元(token)流

词法分析器读入源文件的字符流，将他们组织称有意义的词素(lexeme)序列，对于每个词素，此法分析器产生词法单元（token）作为输出。

```shell
$ xcrun clang -fmodules -fsyntax-only -Xclang -dump-tokens <inputName>.c
```

# 语法分析
语法分析(semantic analysis)
> 词法单元(token)流 -> 抽象语法树(AST)

词法分析的Token流会被解析成一颗抽象语法树(abstract syntax tree - AST)
有了抽象语法树，clang就可以对这个树进行分析，找出代码中的错误。比如**类型不匹配**，亦或Objective C中向target发送了一个**未实现的消息**。

```shell
$ xcrun clang -fsyntax-only -Xclang -ast-dump <inputName>.c | open -f
```

**AST是开发者编写clang插件主要交互的数据结构** ，clang也提供很多API去读取AST。更多细节： [Introduction to the Clang AST](https://clang.llvm.org/docs/IntroductionToTheClangAST.html) 。

# CodeGen
> 抽象语法树(AST) -> LR代码

CodeGen遍历语法树，生成LLVM LR代码。LLVM IR是前端的输出，后端的输入。

Objective-C代码在这一步会进行runtime的桥接：**property合成**，**ARC处理**等。

LLVM会对生成的IR进行优化，优化会调用相应的Pass进行处理。Pass由多个节点组成，都是 [Pass](http://llvm.org/doxygen/classllvm_1_1Pass.html) 类的子类，每个节点负责做特定的优化，更多细节： [Writing an LLVM Pass](https://llvm.org/docs/WritingAnLLVMPass.html) 。

```shell
xcrun clang -S -emit-llvm <inputName>.c -o <outputName>.ll
```

# 生成汇编代码
> LR代码 -> 汇编代码(.s)

LLVM对LR进行优化后，会针对不同架构生成不同的目标代码，最后以汇编代码的格式输出

生成arm 64汇编：
```shell
$ xcrun clang -S <inputName>.c -o <outputName>.s
```

# 汇编器
> 汇编代码(.s) -> 机器代码(.o)

汇编器以汇编代码作为输入，将汇编代码转换为机器代码，最后输出目标文件(object file)。

```shell
$ xcrun clang -fmodules -c <inputName>.c -o <outputName>.o
```

# 链接
> 机器代码(.o) + dylib, a, tbd -> mach-o文件

连接器把编译产生的.o文件和（dylib,a,tbd）文件，生成一个mach-o文件
```shell
$ xcrun clang <inputName>.o -o <outputName>
```

# 例子
从代码层面看一下具体的转化过程，新建一个main.c:

```c
#include <stdio.h>
// 一点注释
#define DEBUG 1
int main() {
#ifdef DEBUG
  printf("hello debug\n");
#else
  printf("hello world\n");
#endif
  return 0;
}
```

[[Clang编译一个C语言文件#过程]]

### [[Clang编译一个C语言文件#预处理]]
执行：
```shell
xcrun clang -E main.c
```

输出：
```shell

...

int main() {
  printf("hello debug\n");
  return 0;
}

```

![[maic_preprocessor.c]]

`#include "stdio.h"` 就是告诉预处理器将这一行替换成头文件 `stdio.h` 中的内容，这个过程是递归的：因为 `stdio.h` 也有可能包含其头文件。

预处理后的文件有400多行，在文件的末尾，可以找到main函数。在预处理的时候，**注释被删除**，**条件编译被处理**。

### [[Clang编译一个C语言文件#词法分析]]

执行：
```shell
$ xcrun clang -fmodules -fsyntax-only -Xclang -dump-tokens main.c
```

输出：
```shell
annot_module_include '#include <stdio.h>
/'		Loc=<main.c:1:1>
int 'int'	 [StartOfLine]	Loc=<main.c:4:1>
identifier 'main'	 [LeadingSpace]	Loc=<main.c:4:5>
....

```

`Loc=<main.c:1:1>` 标示这个token位于源文件main.c的第1行，从第1个字符开始。保存token在源文件中的位置是方便后续clang分析的时候能够找到出错的原始位置。

![[maic_preprocessor 1.c]]

### [[Clang编译一个C语言文件#语法分析]]

执行：
```shell
$ xcrun clang -fsyntax-only -Xclang -ast-dump main.c | open -f
```

输出：
```shell
[0;34m`-[0m[0;1;32mFunctionDecl[0m[0;33m 0x7fcc188dc700[0m <[0;33mmain.c:4:1[0m, [0;33mline:11:1[0m> [0;33mline:4:5[0m[0;1;36m main[0m [0;32m'int ()'[0m
[0;34m  `-[0m[0;1;35mCompoundStmt[0m[0;33m 0x7fcc188dc918[0m <[0;33mcol:12[0m, [0;33mline:11:1[0m>
[0;34m    |-[0m[0;1;35mCallExpr[0m[0;33m 0x7fcc188dc880[0m <[0;33mline:6:3[0m, [0;33mcol:25[0m> [0;32m'int'[0m[0;36m[0m[0;36m[0m
[0;34m    | |-[0m[0;1;35mImplicitCastExpr[0m[0;33m 0x7fcc188dc868[0m <[0;33mcol:3[0m> [0;32m'int (*)(const char *, ...)'[0m[0;36m[0m[0;36m[0m <[0;31mFunctionToPointerDecay[0m>
[0;34m    | | `-[0m[0;1;35mDeclRefExpr[0m[0;33m 0x7fcc188dc7a0[0m <[0;33mcol:3[0m> [0;32m'int (const char *, ...)'[0m[0;36m[0m[0;36m[0m [0;1;32mFunction[0m[0;33m 0x7fcc188c5160[0m[0;1;36m 'printf'[0m [0;32m'int (const char *, ...)'[0m
[0;34m    | `-[0m[0;1;35mImplicitCastExpr[0m[0;33m 0x7fcc188dc8c8[0m <[0;33mcol:10[0m> [0;32m'const char *'[0m[0;36m[0m[0;36m[0m <[0;31mBitCast[0m>
[0;34m    |   `-[0m[0;1;35mImplicitCastExpr[0m[0;33m 0x7fcc188dc8b0[0m <[0;33mcol:10[0m> [0;32m'char *'[0m[0;36m[0m[0;36m[0m <[0;31mArrayToPointerDecay[0m>
[0;34m    |     `-[0m[0;1;35mStringLiteral[0m[0;33m 0x7fcc188dc808[0m <[0;33mcol:10[0m> [0;32m'char [13]'[0m[0;36m lvalue[0m[0;36m[0m[0;1;36m "hello debug\n"[0m
[0;34m    `-[0m[0;1;35mReturnStmt[0m[0;33m 0x7fcc188dc900[0m <[0;33mline:10:3[0m, [0;33mcol:10[0m>
[0;34m      `-[0m[0;1;35mIntegerLiteral[0m[0;33m 0x7fcc188dc8e0[0m <[0;33mcol:10[0m> [0;32m'int'[0m[0;36m[0m[0;36m[0m[0;1;36m 0[0m
```

![[open_esXGnaV3.txt]]


### [[Clang编译一个C语言文件#CodeGen]]

执行：
```shell
xcrun clang -S -emit-llvm main.c -o main.ll
```

输出
```shell
; ModuleID = 'main.c'
source_filename = "main.c"
target datalayout = "e-m:o-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-apple-macosx11.0.0"

@.str = private unnamed_addr constant [13 x i8] c"hello world\0A\00", align 1

; Function Attrs: noinline nounwind optnone ssp uwtable
define i32 @main() #0 {
  %1 = alloca i32, align 4
  store i32 0, i32* %1, align 4
  %2 = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([13 x i8], [13 x i8]* @.str, i64 0, i64 0))
  ret i32 0
}
...

```

![[main.ll]]

### [[Clang编译一个C语言文件#生成汇编代码]]

执行：
```shell
#生成arm 64汇编
$ xcrun clang -S main.c -o main.s
```

输出：
```shell
...
_main:                                  ## @main
        .cfi_startproc
## %bb.0:
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset %rbp, -16
        movq    %rsp, %rbp
...

```
![[main.s]]


### [[Clang编译一个C语言文件#汇编器]]
执行：
```shell
$ xcrun clang -fmodules -c main.c -o main.o
```

还记得我们代码中调用了一个函数 `printf` 么？通过nm命令，查看下main.o中的符号

```shell
$ xcrun nm -nm main.o
                 (undefined) external _printf
0000000000000000 (__TEXT,__text) external _main
```

`_printf` 是一个是undefined external的。undefined表示在当前文件暂时找不到符号 `_printf` ，而external表示这个符号是外部可以访问的，对应表示文件私有的符号是 `non-external` 。

**Tips** ：什么是符号(Symbols)? 符号就是指向一段代码或者数据的名称。还有一种叫做WeakSymols，也就是并不一定会存在的符号，需要在运行时决定。比如iOS 12特有的API，在iOS11上就没有。


### [[Clang编译一个C语言文件#链接]]

执行：
```shell
$ xcrun clang main.o -o main
```

我们就得到了一个mach o格式的可执行文件

```shell
$ file main
main: Mach-O 64-bit executable x86_64
$ ./main 
hello debug
```

在用nm命令，查看可执行文件的符号表：

```shell
$ nm -nm main
                 (undefined) external _printf (from libSystem)
                 (undefined) external dyld_stub_binder (from libSystem)
0000000100000000 (__TEXT,__text) [referenced dynamically] external __mh_execute_header
0000000100000f60 (__TEXT,__text) external _main
```

_printf仍然是 `undefined` ，但是后面多了一些信息： `from libSystem` ，表示这个符号来自于 `libSystem` ，会在运行时动态绑定。