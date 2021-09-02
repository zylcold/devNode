当你在XCode中，选择build的时候（快捷键command+B），会执行如下过程

### 1. 编译信息写入辅助文件，创建编译后的文件架构(name.app)
### 2. 处理文件打包信息，例如在debug环境下
```shell
Entitlements:
{
    "application-identifier" = "app的bundleid";
    "aps-environment" = development;
}
```

### 3. 执行CocoaPod编译前脚本
例如对于使用CocoaPod的工程会执行 `CheckPods Manifest.lock`

### 4. 编译各个.m文件，使用 `CompileC` 和 `clang` 命令。

```shell
CompileC ClassName.o ClassName.m normal x86_64 objective-c com.apple.compilers.llvm.clang.1_0.compiler

export LANG=en_US.US-ASCII
export PATH="..."

clang -x objective-c -arch x86_64 -fmessage-length=0 -fobjc-arc... -Wno-missing-field-initializers ... -DDEBUG=1 ... -isysroot iPhoneSimulator10.1.sdk -fasm-blocks ... -I 上文提到的文件 -F 所需要的Framework  -iquote 所需要的Framework  ... -c ClassName.c -o ClassName.o

```

通过这个编译的命令，我们可以看到

```shell
clang是实际的编译命令
-x 					
	objective-c 指定了编译的语言
	
-arch 				
	x86_64制定了编译的架构，类似还有arm7等
	
-fobjc-arc 			
	一些列-f开头的，指定了采用arc等信息。这个也就是为什么你可以对单独的一个.m文件采用非ARC编程。
	
-Wno-missing-field-initializers 
	一系列以-W开头的，指的是编译的警告选项，通过这些你可以定制化编译选项
	
-DDEBUG=1 
	一些列-D开头的，指的是预编译宏，通过这些宏可以实现条件编译
	
-iPhoneSimulator10.1.sdk 
	制定了编译采用的iOS SDK版本
	
-I 
	把编译信息写入指定的辅助文件
	
-F 
	链接所需要的Framework
	
-c 
	ClassName.c 编译文件
	
-o 
	ClassName.o 编译产物
```

### 5. 链接需要的Framework
	例如 `Foundation.framework`,`AFNetworking.framework`,`ALiPay.fframework`

### 6. 编译xib文件
### 7. 拷贝xib，图片等资源文件到结果目录
### 8. 编译ImageAssets
### 9. 处理info.plist
### 10. 执行CocoaPod脚本
### 11. 拷贝Swift标准库
### 12. 创建.app文件
### 13. 签名