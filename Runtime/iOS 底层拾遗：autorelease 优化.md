#runtime 

# 前言

听闻 ARC 下 autorelease 操作有一些优化，总感觉云里雾里的，笔者初略的探究了一番，记录下来变成这篇水文。

由于 ARC 下 `retain/release/autorelease` 的调用都是编译器代劳，所以需要使用编译后的代码进行分析，通常笔者选择 Xcode 自带的工具，它有一个优势是自动将一些符号地址改为符号名，并且可以选择 Running 或 Archiving 下的汇编代码，后者生成的代码往往是前者的优化版本。

本文基于 Runtime 750，arm64，Archiving 汇编代码。

先前置声明一个后文会用到的类：

```objc
@interface TestObject : NSObject <NSCopying>
@end

@implementation TestObject

+ (id)foo {
	return [NSObject new];
}

- (id)copyWithZone:(NSZone *)zone {
	return [NSObject new];
}

@end

```

# 一、alloc / new / copy / mutableCopy 方法

编写这样一段代码：

```objc

[[NSObject new] copy];

```

汇编代码为：

```assembly

...
	adrp	x8, l_OBJC_CLASSLIST_REFERENCES_$_@PAGE
	ldr	x0, [x8, l_OBJC_CLASSLIST_REFERENCES_$_@PAGEOFF]
	adrp	x8, l_OBJC_SELECTOR_REFERENCES_@PAGE
	ldr	x1, [x8, l_OBJC_SELECTOR_REFERENCES_@PAGEOFF]
	bl	_objc_msgSend
	mov	x19, x0
	adrp	x8, l_OBJC_SELECTOR_REFERENCES_.6@PAGE
	ldr	x1, [x8, l_OBJC_SELECTOR_REFERENCES_.6@PAGEOFF]
	bl	_objc_msgSend
	bl	_objc_release
	mov	x0, x19
	bl	_objc_release
...


```

`...@PAGE / ...@PAGEOFF` 两句一起出现表示通过页基址+符号在页中偏移来计算地址。首先把 `NSObject` 类地址和 `new` 方法地址找到分别放入 `x0` 和 `x1` ，然后调用 `_objc_msgSend` ，调用完成后 `x0` 里面放的就是 `[NSObject new]` 得到的对象地址，所以后面直接找到 `copy` 方法调用。这一段基本分析后文不会再详细描述了。

由于 `new` 和 `copy` 将引用计数+2，其后调用了两次 `_objc_release` ，这很符合直觉，看起来编译器并未做什么优化。

尝试使用 `alloc` 和 `mutableCopy` ，得到几乎一致的结果，似乎就能得到只要是生成本类实例的方法都不会做优化的结论？
**并不能。**

将上面的代码换成：

```objc
[[TestObject new] copy];
```

发现编译后的汇编代码仍然和上面的差不多，而此时 `TestObject` 的 `copy` 返回了另外类的实例，退一步讲，前面的 `NSObject` 的 `copy` 方法也并未实现，所以可以猜测：

**编译器不是通过返回类型来判断的，而是通过简单的符号匹配，发现`alloc/new/copy/mutableCopy` 符号就不做优化。**

# 二、自定义带返回值的方法

写这样一句代码：

```objc
[TestObject foo];
```

汇编代码为：

```objc
...
	bl	_objc_msgSend
Ltmp9:
	mov	x29, x29	; marker for objc_retainAutoreleaseReturnValue
	bl	_objc_unsafeClaimAutoreleasedReturnValue
...

```

## 调用方的逻辑

调用 `_objc_msgSend` 前 `x0 -> TestObject, x1 -> foo` ，这里出现了一个不符合直觉的 C 函数 `_objc_unsafeClaimAutoreleasedReturnValue` ，光看名字猜不到具体是干嘛的，所以去掉 C 语言符号修饰 `_` ，直接查看源码：

```objc
id objc_unsafeClaimAutoreleasedReturnValue(id obj) {
    if (acceptOptimizedReturn() == ReturnAtPlus0) return obj;
    return objc_releaseAndReturn(obj);
}

```

`objc_releaseAndReturn` 函数不展开，只需知道是真正的 `release` 操作。那么大家就奇怪了，理论上 `foo` 方法会把返回值先加入自动释放池，这里根本就不需要 `release` ，常理来说只有当这个 `acceptOptimizedReturn() == ReturnAtPlus0` 为 YES 不做 `release` 操作才与大家的意识契合。那么看一下这个关键方法：


```objc
enum ReturnDisposition : bool {
    ReturnAtPlus0 = false, ReturnAtPlus1 = true
};

static ALWAYS_INLINE ReturnDisposition acceptOptimizedReturn() {
    ReturnDisposition disposition = getReturnDisposition();
    setReturnDisposition(ReturnAtPlus0);  // reset to the unoptimized state
    return disposition;
}

static ALWAYS_INLINE ReturnDisposition getReturnDisposition() {
    return (ReturnDisposition)(uintptr_t)tls_get_direct(RETURN_DISPOSITION_KEY);
}

static ALWAYS_INLINE void setReturnDisposition(ReturnDisposition disposition) {
    tls_set_direct(RETURN_DISPOSITION_KEY, (void*)(uintptr_t)disposition);
}
```

看到最终是使用 `tls_get_direct(RETURN_DISPOSITION_KEY)` 取的 TLS（Thread Local Storage）里的值，取了后还把这个值变为 `false` 。看起来这里的代码非常简单，我们只需要关心是 **何时把 TLS 对应的值变成`true` 的** 。


## 被调用方的逻辑

由于刚才分析了， `foo` 函数会将生成的对象放入自动释放池，这里理论上不需要 `release` ，直接分析汇编代码看是否有什么奇怪操作：

```c
...
	bl	_objc_msgSend
	ldp	x29, x30, [sp], #16
	b	_objc_autoreleaseReturnValue
...

```

发现调用了 `_objc_autoreleaseReturnValue` 函数，切到源码：

```objc
id objc_autoreleaseReturnValue(id obj) {
    if (prepareOptimizedReturn(ReturnAtPlus1)) return obj;
    return objc_autorelease(obj);
}
static ALWAYS_INLINE bool prepareOptimizedReturn(ReturnDisposition disposition) {
    assert(getReturnDisposition() == ReturnAtPlus0);
    if (callerAcceptsOptimizedReturn(__builtin_return_address(0))) {
        if (disposition) setReturnDisposition(disposition);
        return true;
    }
    return false;
}

```

`objc_autorelease` 就是真正的 `autorelease` 操作，会将对象加入自动释放池。若 `if` 判断成功会调用了一个前面分析过的方法 `setReturnDisposition(...)` 将 TLS 对应 Key 的值设置为 `true` ，如果设置成功后将放弃 `autorelease` 操作。

## 核心逻辑分析

这里先考虑做了优化的情况。

**重点分析：** 若这里不进行 `autorelease` ，调用 `foo` 后生成的对象将不会被自动释放池管理，这个对象的引用计数为 1。那之前的 `objc_unsafeClaimAutoreleasedReturnValue(...)` 函数就需要进行 `release` 操作， `[TestObject foo]` 生成对象的引用计数才能减为 0。对比前面分析完全符合，至此，形成了逻辑通路。

**优化点：** 这个场景少了一步 `autorelease` ，多了一步 `release` ，也就是省去了加入自动释放池的时间消耗、避免对象对自动释放池的内存占用。

另外一个场景，代码如下：

```objc
id obj = [TestObject foo];
```

汇编代码有些变化：

```c
...
	bl	_objc_msgSend
	mov	x29, x29	; marker for objc_retainAutoreleaseReturnValue
	bl	_objc_retainAutoreleasedReturnValue
	bl	_objc_release
...

```

笔者只是加了一个临时变量持有这个返回值，直觉上会调用 `_objc_retain` 然后调用 `_objc_release` ，但先调用的是 `_objc_retainAutoreleasedReturnValue` ：

```objc
id objc_retainAutoreleasedReturnValue(id obj) {
    if (acceptOptimizedReturn() == ReturnAtPlus1) return obj;
    return objc_retain(obj);
}

```

`foo` 方法若不会将创建的对象进行 `autorelease` ，那么这里也就不需要进行 `retain` 了，很好理解。

**优化点：** 这个场景少了一步 `autorelease` ，少了一步 `retain` ，优化效果就变得明显了。

## 如何判断 autorelease 是否需要优化？

看关键的一个判断：

```c
if (callerAcceptsOptimizedReturn(__builtin_return_address(0))) ...

```

`__builtin_return_address(n)` 拿到的是前第 n + 1 函数栈的 `lr` (Link Register) ，那么这里就是拿到上一个栈的 `lr` （函数在调用其它函数时会将下一句指令地址写入 `lr` 便于恢复执行），也就是前面的这句汇编代码（别疑惑函数栈这么深怎么会拿到第一个函数的 `lr` ，因为这些函数都是内联的）：

```
mov x29, x29    ; marker for objc_retainAutoreleaseReturnValue
```

看起来这句代码什么也没做，先看一下这个判断函数：

```c
static ALWAYS_INLINE bool callerAcceptsOptimizedReturn(const void *ra) {
    // fd 03 1d aa    mov fp, fp
    // arm64 instructions are well-aligned
    if (*(uint32_t *)ra == 0xaa1d03fd) {
        return true;
    }
    return false;
}

```

这个函数就是将 `ra` 指向的值和 `0xaa1d03fd` 对比，若相等就执行优化。注释已经比较明白了，这个值就是 `mov fp, fp` 的十六进制表示（注意 iOS 是小端）。

那么只要被调用方拿到调用方的 `lr` 判断就行了，所以是否进行这个优化的决定权在调用方手中。

## MRC 模式下是否开启了优化

将代码设置为 `-fno-objc-arc` ，随便写些代码查看汇编，发现编译器并不会将开发者显式写的 `objc_autorelease/objc_retain/objc_release` 函数强制改为 `objc_autoreleaseReturnValue` 等方法，且 MRC 下编译的代码在调用方法时不会加入 `mov fp fp` 企图优化（推理也可知，因为 `retain/release` 操作是不能优化的）。

`objc_retain/objc_release` 就是直接进行引用计数加减，而 `objc_autorelease` 在 `OBJC2` 上的定义却有所变化：

```c
__attribute__((aligned(16))) id objc_autorelease(id obj) {
    if (!obj) return obj;
    if (obj->isTaggedPointer()) return obj;
    return obj->autorelease();
}

inline id objc_object::autorelease() {
    if (isTaggedPointer()) return (id)this;
    if (fastpath(!ISA()->hasCustomRR())) return rootAutorelease();
    return ((id(*)(objc_object *, SEL))objc_msgSend)(this, SEL_autorelease);
}

inline id objc_object::rootAutorelease() {
    if (isTaggedPointer()) return (id)this;
    if (prepareOptimizedReturn(ReturnAtPlus1)) return (id)this;
    return rootAutorelease2();
}

```

若当前类没有自定义的 `retain/release` 方法实现时，最终仍然会调用 `prepareOptimizedReturn(ReturnAtPlus1)` 进行优化，所以当调用方是 ARC，被调用方是 MRC 时这个优化仍然有效。

## 为什么要双方协商 autorelease 优化？

**总结一下：** 不管被调用方是 MRC 还是 ARC，进行 `autorelease` 操作时都会尝试去优化，但是只有调用方是 ARC 时才能优化成功。

那么协商的意义就很重要了，这样才能保证版本兼容以及 MRC 和 ARC 的兼容。

## 为什么使用线程局部存储？

一个线程可以理解为一个一个顺序执行指令的，那么用一个 bool 值就能解决线程执行的所有代码的优化。

TLS 是线程私有的，所以 autorelease 的优化是线程间互不影响的，引用计数的加减线程安全由相应的函数保证。若这个优化的控制是多线程共同制约的，那么单纯一个 bool 值是实现不了的，需要更复杂的数据结构，并且要额外处理线程并发的安全问题，这些工作反而会降低程序执行的效率。

# 后语

本文通过探索的方式分析了 autorelease 的优化逻辑，实际上并不能铁板钉钉的说明事实，只有通过查看 clang 编译器代码才能真正的有说服力。不过从理解原理的角度来说按图索骥是个非常好的学习方式，希望本文能给读者朋友带来一些帮助。

[iOS 底层拾遗：autorelease 优化](https://juejin.cn/post/6844903971497197576)