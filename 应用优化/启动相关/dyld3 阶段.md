#launch #optimize 
# 创建启动闭包

dyld 会首先创建启动闭包，闭包是一个缓存，用来提升启动速度的。既然是缓存，那么必然不是每次启动都创建的，只有在重启手机或者更新/下载 App 的第一次启动才会创建。 **闭包存储在沙盒的 tmp/com.apple.dyld 目录，清理缓存的时候切记不要清理这个目录** 。

闭包是怎么提升启动速度的呢？我们先来看一下闭包里都有什么内容：

* dependends，依赖动态库列表
* fixup：bind & rebase 的地址
* initializer-order：初始化调用顺序
* optimizeObjc: Objective C 的元数据
* 其他：main entry, uuid…

动态库的依赖是树状的结构，初始化的调用顺序是先调用树的叶子结点，然后一层层向上，最先调用的是 libSystem，因为他是所有依赖的源头。

![[c98498fff55540edaf8c01c522183bc0~tplv-k3u1fbpfcp-zoom-1.image.png]]


为什么闭包能提高启动速度呢？

**因为这些信息是每次启动都需要的，把信息存储到一个缓存文件就能避免每次都解析，尤其是 Objective C 的运行时数据（Class/Method...）解析非常慢** 。

# fixup

有了闭包之后，就可以用闭包启动 App 了。这时候很多动态库还没有加载进来，会首先对这些动态库 mmap 加载到虚拟内存里。接着会对每个 Mach-O 做 fixup，包括 Rebase 和 Bind。

* **Rebase：修复内部指针** 。这是因为 Mach-O 在 mmap 到虚拟内存的时候，起始地址会有一个随机的偏移量 slide，需要把内部的指针指向加上这个 slide。
* **Bind：修复外部指针** 。这个比较好理解，因为像 printf 等外部函数，只有运行时才知道它的地址是什么，bind 就是把指针指向这个地址。

举个例子：一个 Objective C 字符串@"1234"，编译到最后的二进制的时候是会存储在两个 section 里的

* `__TEXT，__cstring` ，存储实际的字符串"1234"
* `__DATA，__cfstring` ，存储 Objective C 字符串的元数据，每个元数据占用 32Byte，里面有两个指针：内部指针，指向 `__TEXT，__cstring` 中字符串的位置；外部指针 isa，指向类对象的，这就是为什么可以对 Objective C 的字符串字面量发消息的原因。

如下图，编译的时候，字符串 1234 在 `__cstring` 的 0x10 处，所以 DATA 段的指针指向 0x10。但是 mmap 之后有一个偏移量 slide=0x1000，这时候字符串在运行时的地址就是 0x1010，那么 DATA 段的指针指向就不对了。 **Rebase 的过程就是把指针从 0x10，加上 slide 变成 0x1010。运行时类对象的地址已经知道了，bind 就是把 isa 指向实际的内存地址** 。

# LibSystem Initializer

Bind & Rebase 之后，首先会执行 LibSystem 的 Initializer，做一些最基本的初始化：

* 初始化 libdispatch
* 初始化 objc runtime，注册 sel，加载 category

注意这里没有初始化 objc 的类方法等信息，是因为启动闭包的缓存数据已经包含了 optimizeObjc。

# Load & Static Initializer

接下来会进行 main 函数之前的一些初始化，主要包括+load 和 static initializer。这两类初始化函数都有个特点： **调用顺序不确定，和对应文件的链接顺序有关系** 。那么就会存在一个隐藏的坑：有些注册逻辑在+load 里，对应会有一些地方读取这些注册的数据，如果在+load 中读取，很有可能读取的时候还没有注册。

那么，如何找到代码里有哪些 load 和 static initializer 呢？

在 Build Settings 里可以配置 write linkmap，这样在生成的 linkmap 文件里就可以找到有哪些文件里包含 load 或者 static initializer：

* `__mod_init_func` ，static initializer
* `__objc_nlclslist` ，实现+load 的类
* `__objc_nlcatlist` ，实现+load 的 Category

# load 举例

如果+load 方法里的内容很简单，会影响启动时间么？比如这样的一个+load 方法？

```
+ (void)load 
{
    printf("1234");
}

```

编译完了之后，这个函数会在二进制中的 TEXT 两个段存在： `__text` **存函数二进制，** `cstring` **存储字符串 1234** 。为了执行函数，首先要访问 `__text` 触发一次 Page In 读入物理内存，为了打印字符串，要访问 `__cstring` ，还会触发一次 Page In。

* 为了执行这个简单的函数， **系统要额外付出两次 Page In 的代价** ，所以 load 函数多了，page in 会成为启动性能的瓶颈。

# static initializer 产生的条件

静态初始化是从哪来的呢？以下几种代码会导致静态初始化

* `__attribute__((constructor))`
* `static class object`
* `static object in global namespace`

注意，并不是所有的 static 变量都会产生静态初始化，编译器很智能，对于在编译期间就能确定的变量是会直接 inline。

```
//会产生静态初始化
class Demo{ 
static const std::string var_1; 
};
const std::string var_2 = "1234"; 
static Logger logger;
//不会产生静态初始化
static const int var_3 = 4; 
static const char * var_4 = "1234";
复制代码
```

std::string 会合成 static initializer 是因为初始化的时候必须执行构造函数，这时候编译器就不知道怎么做了，只能延迟到运行时～