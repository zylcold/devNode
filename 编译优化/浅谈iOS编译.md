#build 


* Github地址： [LeoMobileDeveloper](https://github.com/LeoMobileDeveloper)

# 前言

一般可以将编程语言分为两种， [编译语言](https://zh.wikipedia.org/wiki/%E7%B7%A8%E8%AD%AF%E8%AA%9E%E8%A8%80) 和 [直译式语言](https://en.wikipedia.org/wiki/Interpreted_language)。

像C++,Objective C都是编译语言。编译语言在执行的时候，必须先通过编译器生成机器码，机器码可以直接在CPU上执行，所以执行效率较高。

像JavaScript,Python都是直译式语言。直译式语言不需要经过编译的过程，而是在执行的时候通过一个中间的解释器将代码解释为CPU可以执行的代码。所以，较编译语言来说，直译式语言效率低一些，但是编写的更灵活，也就是为啥JS大法好。

iOS开发目前的常用语言是：Objective和Swift。二者都是编译语言，换句话说都是需要编译才能执行的。二者的编译都是依赖于Clang(swift) + LLVM. 篇幅限制，本文只关注Objective C，因为原理上大同小异。

---

[[编译器基本介绍]]

# iOS编译

可以通过Clang，来查看一个文件的编译具体过程，新建Demo.m

```c
#import <Foundation/Foundation.h>
  
int main(){
    @autoreleasepool {
        NSLog(@"%@",@"Hello Leo");
    }
    return 0;
}
```

然后终端输入：

```shell
clang -ccc-print-phases -framework Foundation Demo.m -o Demo 

0: input, "Foundation", object 
1: input, "Demo.m", objective-c
2: preprocessor, {1}, objective-c-cpp-output//预处理
3: compiler, {2}, ir //编译生成IR(中间代码)
4: backend, {3}, assembler//汇编器生成汇编代码
5: assembler, {4}, object//生成机器码
6: linker, {0, 5}, image//链接
7: bind-arch, "x86_64", {6}, image//生成Image，也就是最后的可执行文件
```

接着，就可以在终端直接运行这个程序了：

```shell
./Demo
Leo$ ./Demo 
Demo[923:24816] Hello Leo
```

---

# [[Xcode build的流程]]

---

# [[IPA包的内容]]

---

# [[符号表dSYM]]

---

# 那些你想到和想不到的应用场景

### `__attribute__`
或多或少，你都会在第三方库或者iOS的头文件中，见到过__attribute__。
比如

```c
__attribute__ ((warn_unused_result)) //如果没有使用返回值，编译的时候给出警告
```

`__attribtue__` 是一个高级的的编译器指令，它允许开发者指定更更多的编译检查和一些高级的编译期优化。
分为三种：
* 函数属性 （Function Attribute）
* 类型属性 (Variable Attribute )
* 变量属性 (Type Attribute )

语法结构
```c
__attribute__ ((attribute-list))
```
放在声明分号“;”前面。

比如，在三方库中最常见的，声明一个属性或者方法在当前版本弃用了
```c
@property (strong,nonatomic) CLASSNAME * property __deprecated;

```

这样的好处是：给开发者一个过渡的版本，让开发者知道这个属性被弃用了，应当使用最新的API，但是被__deprecated的属性仍然可以正常使用。如果直接弃用，会导致开发者在更新Pod的时候，代码无法运行了。

`__attribtue__` 的使用场景很多，本文只列举iOS开发中常用的几个：

```c
//弃用API，用作API更新
#define __deprecated	__attribute__((deprecated)) 

//带描述信息的弃用
#define __deprecated_msg(_msg) __attribute__((deprecated(_msg)))

//遇到__unavailable的变量/方法，编译器直接抛出Error
#define __unavailable	__attribute__((unavailable))

//告诉编译器，即使这个变量/方法 没被使用，也不要抛出警告
#define __unused	__attribute__((unused))

//和__unused相反
#define __used		__attribute__((used))

//如果不使用方法的返回值，进行警告
#define __result_use_check __attribute__((__warn_unused_result__))

//OC方法在Swift中不可用
#define __swift_unavailable(_msg)	__attribute__((__availability__(swift, unavailable, message=_msg)))

```

### Clang警告处理
你一定还见过如下代码：
```
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wundeclared-selector"
///代码
#pragma clang diagnostic pop

```

这段代码的作用是
1. 对当前编译环境进行压栈
2. 忽略 `-Wundeclared-selector` （未声明的）Selector警告
3. 编译代码
4. 对编译环境进行出栈

通过clang diagnostic push/pop,你可以灵活的控制代码块的编译选项。

我在之前的一篇文章里，详细的介绍了XCode的警告相关内容。本文篇幅限制，就不详细讲解了。

* [iOS 合理利用Clang警告来提高代码质量](http://blog.csdn.net/Hello_Hwc/article/details/46425503)

### 预处理
所谓预处理，就是在编译之前的处理。预处理能够让你定义编译器变量，实现条件编译。
比如，这样的代码很常见

```
#ifdef DEBUG
//...
#else
//...
#endif
```

同样，我们同样也可以定义其他预处理变量,在XCode-选中Target-build settings中，搜索proprecess。然后点击图中蓝色的加号，可以分别为debug和release两种模式设置预处理宏。
比如我们加上： `TestServer` ，表示在这个宏中的代码运行在测试服务器

![[20161205221447674.jpeg]]

然后，配合多个Target（右键Target，选择Duplicate），单独一个Target负责测试服务器。这样我们就不用每次切换测试服务器都要修改代码了。

```
#ifdef TESTMODE
//测试服务器相关的代码
#else
//生产服务器相关代码
#endif
12345
```

### 插入脚本

通常，如果你使用CocoaPod来管理三方库，那么你的Build Phase是这样子的：

![[20161205223019180.jpeg]]

其中：[CP]开头的，就是CocoaPod插入的脚本。

* Check Pods Manifest.lock，用来检查cocoapod管理的三方库是否需要更新
* Embed Pods Framework，运行脚本来链接三方库的静态/动态库
* Copy Pods Resources，运行脚本来拷贝三方库的资源文件

而这些配置信息都存储在这个文件(.xcodeprog)里

![[20161205224255436.jpeg]]

到这里，CocoaPod的原理也就大致搞清楚了，通过修改xcodeproject，然后配置编译期脚本，来保证三方库能够正确的编译连接。

同样，我们也可以插入自己的脚本，来做一些额外的事情。比如，每次进行archive的时候，我们都必须手动调整target的build版本，如果一不小心，就会忘记。这个过程，我们可以通过插入脚本自动化。

```
buildNumber=$(/usr/libexec/PlistBuddy -c "Print CFBundleVersion" "${PROJECT_DIR}/${INFOPLIST_FILE}")
buildNumber=$(($buildNumber + 1))
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion $buildNumber" "${PROJECT_DIR}/${INFOPLIST_FILE}"
```

这段脚本其实很简单，读取当前pist的build版本号,然后对其加一，重新写入。

使用起来也很简单：

* Xcode - 选中Target - 选中build phase
* 选择添加Run Script Phase

![[20161210104221850.jpeg]]

* 然后把这段脚本拷贝进去，并且勾选Run Script Only When installing，保证只有我们在安装到设备上的时候，才会执行这段脚本。重命名脚本的名字为Auto Increase build number

![[_20161210104221850.jpeg]]

* 然后，拖动这个脚本的到Link Binary With Libraries下面

![[20161210104750399.jpeg]]

### 脚本编译打包

脚本化编译打包对于CI（持续集成）来说，十分有用。iOS开发中，编译打包必备的两个命令是：

```
//编译成.app
xcodebuild  -workspace $projectName.xcworkspace -scheme $projectName  -configuration $buildConfig clean build SYMROOT=$buildAppToDir

//打包
xcrun -sdk iphoneos PackageApplication -v $appDir/$projectName.app -o $appDir/$ipaName.ipa

//通过info命令，可以查看到详细的文档
info xcodebuild
```

在本文最后的附录中，提供了我之前使用的一个自动打包的脚本。

## 提高项目编译速度

通常，当项目很大，源代码和三方库引入很多的时候，我们会发现编译的速度很慢。在了解了XCode的编译过程后，我们可以从以下角度来优化编译速度：

### 查看编译时间

我们需要一个途径，能够看到编译的时间，这样才能有个对比，知道我们的优化究竟有没有效果。
对于XCode 8，关闭XCode，终端输入以下指令

```
$ defaults write com.apple.dt.Xcode ShowBuildOperationDuration YES
```

然后，重启XCode，然后编译，你会在这里看到编译时间。

![[20161206223153632.jpeg]]

### 代码层面的优化
#### forward declaration
所谓 `forward declaration` ，就是 `@class CLASSNAME` ，而不是 `#import CLASSNAME.h` 。这样，编译器能大大提高#import的替换速度。

#### 对常用的工具类进行打包（Framework/.a）
打包成Framework或者静态库，这样编译的时候这部分代码就不需要重新编译了。

#### 常用头文件放到预编译文件里
XCode的pch文件是预编译文件，这里的内容在执行XCode build之前就已经被预编译，并且引入到每一个.m文件里了。

### 编译器选项优化
#### Debug模式下，不生成dsym文件
上文提到了，dysm文件里存储了调试信息，在Debug模式下，我们可以借助XCode和LLDB进行调试。所以，不需要生成额外的dsym文件来降低编译速度。

#### Debug开启 `Build Active Architecture Only`
在XCode -> Build Settings -> Build Active Architecture Only 改为YES。

这样做可以只编译当前的版本，比如arm7/arm64等等，记得只开启Debug模式。这个选项在高版本的XCode中自动开启了。

#### Debug模式下，关闭编译器优化
编译器优化
![[20161206231413542.jpeg]]

---

##附录
自动编译打包脚本

```shell
export LC_ALL=zh_CN.GB2312;
export LANG=zh_CN.GB2312
buildConfig="Release" //这里是build模式
projectName=`find . -name *.xcodeproj | awk -F "[/.]" '{print $(NF-1)}'`
projectDir=`pwd`
wwwIPADir=~/Desktop/$projectName-IPA
isWorkSpace=true

echo "~~~~~~~~~~~~~~~~~~~开始编译~~~~~~~~~~~~~~~~~~~"
if [ -d "$wwwIPADir" ]; then
	echo $wwwIPADir
	echo "文件目录存在"
else
	echo "文件目录不存在"
	mkdir -pv $wwwIPADir
	echo "创建${wwwIPADir}目录成功"
fi

cd $projectDir
rm -rf ./build
buildAppToDir=$projectDir/build
infoPlist="$projectName/Info.plist"
bundleVersion=`/usr/libexec/PlistBuddy -c "Print CFBundleShortVersionString" $infoPlist`
bundleIdentifier=`/usr/libexec/PlistBuddy -c "Print CFBundleIdentifier" $infoPlist`
bundleBuildVersion=`/usr/libexec/PlistBuddy -c "Print CFBundleVersion" $infoPlist`

if $isWorkSpace ; then  #是否用CocoaPod
	echo  "开始编译workspace...."
	xcodebuild  -workspace $projectName.xcworkspace -scheme $projectName  -configuration $buildConfig clean build SYMROOT=$buildAppToDir
else
	echo  "开始编译target...."
	xcodebuild  -target  $projectName  -configuration $buildConfig clean build SYMROOT=$buildAppToDir
fi

if test $? -eq 0 then
	echo "~~~~~~~~~~~~~~~~~~~编译成功~~~~~~~~~~~~~~~~~~~"
else
	echo "~~~~~~~~~~~~~~~~~~~编译失败~~~~~~~~~~~~~~~~~~~"
	exit 1
fi

ipaName=`echo $projectName | tr "[:upper:]" "[:lower:]"` #将项目名转小写
findFolderName=`find . -name "$buildConfig-*" -type d |xargs basename` #查找目录
appDir=$buildAppToDir/$findFolderName/  #app所在路径
echo "开始打包$projectName.app成$projectName.ipa....."
xcrun -sdk iphoneos PackageApplication -v $appDir/$projectName.app -o $appDir/$ipaName.ipa

if [ -f "$appDir/$ipaName.ipa" ] then
	echo "打包$ipaName.ipa成功."
else
	echo "打包$ipaName.ipa失败."
	exit 1
fi

path=$wwwIPADir/$projectName$(date +%Y%m%d%H%M%S).ipa
cp -f -p $appDir/$ipaName.ipa $path   #拷贝ipa文件
echo "复制$ipaName.ipa到${wwwIPADir}成功"
echo "~~~~~~~~~~~~~~~~~~~结束编译，处理成功~~~~~~~~~~~~~~~~~~~"

```

[iOS编译过程的原理和应用_Leo的专栏-CSDN博客](https://blog.csdn.net/Hello_Hwc/article/details/53557308)