#build #clang

## 传统的劣势
传统的 `#include/#import` 都是文本语义：预处理器在处理的时候会把这一行替换成对应头文件的文本，这种简单粗暴替换是有很多问题的：

1. 大量的预处理消耗。假如有N个头文件，每个头文件又 `#include` 了M个头文件，那么整个预处理的消耗是 **N*M** 。
2. 文件导入后，宏定义容易出现问题。因为是文本导入，并且按照include依次替换，当一个头文件定义了 `#define std hello_world` ，而第另一个个头文件刚好又是C++标准库，那么 `include` 顺序不通，可能会导致所有的std都会被替换。
3. 边界不明显。拿到一组.a和.h文件，很难确定.h是属于哪个.a的，需要以什么样的顺序导入才能正确编译。


[clang module](https://clang.llvm.org/docs/Modules.html) 不再使用文本模型，而是采用更高效的语义模型。clang module提供了一种新的导入方式:`@import` ，module会被作为一个独立的模块编译，并且产生独立的缓存，从而大幅度提高预处理效率，这样时间消耗从 **M*N** 变成了 **M+N** 。

XCode创建的Target是Framework的时候，默认define module会设置为YES，从而支持module，当然像Foundation等系统的framwork同样支持module。

`#import <Foundation/NSString.h>` 的时候，编译器会检查 `NSString.h` 是否在一个module里，如果是的话，这一行会被替换成 `@import Foundation` 。

![[image_19.png]]

那么，如何定义一个module呢？答案是：modulemap文件，这个文件描述了一组头文件如何转换为一个module，举个例子：
```c
framework module Foundation  [extern_c] [system] {
	umbrella header "Foundation.h" // 所有要暴露的头文件
 	export *
	module * {
 		export *
 	}
 	explicit module NSDebug { //submodule
 		header "NSDebug.h"
 		export *
 	}
 }
```

swift是可以直接 `import` 一个clang module的，比如你有一些C库，需要在Swift中使用，就可以用modulemap的方式。
