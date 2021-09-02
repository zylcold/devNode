#runtime

OC作为一门面向对象的语言，那么对于对象的创建方法的探索流程就必不可少。下面我们就探索一下关于对象在创建时开辟内存的alloc方法的流程。

## 一、源码

探索之前我们需要一份最新的objc4-781官方源码进行调试，可参考cooci最新的objc4-779.1源码编译调试方法进行调试 [objc官方源码](https://opensource.apple.com/source/objc4/) [老司机最新macOS 10.15下objc4-779.1源码编译调试](https://juejin.im/post/6844904082226806792)


## alloc源码调试

有了源码和知道了调试方式之后，便来到了第三步——alloc源码调试

### 1. alloc方法

```objc
+ (id)alloc {
    return _objc_rootAlloc(self);
}
```

##### 2. _objc_rootAlloc方法

```objc
_objc_rootAlloc(Class cls)
{
    return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}

```

> fastpath表示条件更可能成立 slowpath表示条件更不可能成立

其实将 `fastpath` 和 `slowpath` 去掉是完全不影响任何功能的。之所以将 `fastpath` 和 `slowpath` 放到 `if语句` 中，是为了告诉编译器， `if` 中的条件是大概率 `fastpath` 还是小概率 `slowpath` 事件，从而让编译器对代码进行优化。知道了这些，我们就可以来继续看源码了：

### 3. callAlloc方法

```objc
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
#if __OBJC2__
    if (slowpath(checkNil && !cls)) return nil;
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
        return _objc_rootAllocWithZone(cls, nil);
    }
#endif

    // No shortcuts available.
    if (allocWithZone) {
        return ((id(*)(id, SEL, struct _NSZone *))objc_msgSend)(cls, @selector(allocWithZone:), nil);
    }
    return ((id(*)(id, SEL))objc_msgSend)(cls, @selector(alloc));
}

```

### 4. _objc_rootAllocWithZone方法

```objc
_objc_rootAllocWithZone(Class cls, malloc_zone_t *zone __unused)
{
    // allocWithZone under __OBJC2__ ignores the zone parameter
    return _class_createInstanceFromZone(cls, 0, nil,
                                         OBJECT_CONSTRUCT_CALL_BADALLOC);
}

```

### 5. _class_createInstanceFromZone方法

```objc
static ALWAYS_INLINE id
_class_createInstanceFromZone(Class cls, size_t extraBytes, void *zone,
                              int construct_flags = OBJECT_CONSTRUCT_NONE,
                              bool cxxConstruct = true,
                              size_t *outAllocatedSize = nil)
{
    ASSERT(cls->isRealized());

    // Read class's info bits all at once for performance
    bool hasCxxCtor = cxxConstruct && cls->hasCxxCtor();
    bool hasCxxDtor = cls->hasCxxDtor();
    bool fast = cls->canAllocNonpointer();
    size_t size;

    size = cls->instanceSize(extraBytes);
    if (outAllocatedSize) *outAllocatedSize = size;

    id obj;
    if (zone) {
        obj = (id)malloc_zone_calloc((malloc_zone_t *)zone, 1, size);
    } else {
        // alloc 开辟内存的地方
        obj = (id)calloc(1, size);
    }
    if (slowpath(!obj)) {
        if (construct_flags & OBJECT_CONSTRUCT_CALL_BADALLOC) {
            return _objc_callBadAllocHandler(cls);
        }
        return nil;
    }

    if (!zone && fast) {
        obj->initInstanceIsa(cls, hasCxxDtor);
    } else {
        // Use raw pointer isa on the assumption that they might be
        // doing something weird with the zone or RR.
        obj->initIsa(cls);
    }

    if (fastpath(!hasCxxCtor)) {
        return obj;
    }

    construct_flags |= OBJECT_CONSTRUCT_FREE_ONFAILURE;
    return object_cxxConstructFromClass(obj, cls, construct_flags);
}
```

### 6. cls->instanceSize & calloc & obj->initInstanceIsa

**① cls->instanceSize：先计算出需要的内存空间大小**

1. 首先判断是否有缓存，有的话采用内存对齐方法计算所需内存大小
2. 如果没有缓存，则计算内存大小，如果size 小于 16，最小取16


```objc
size_t instanceSize(size_t extraBytes) const {
    if (fastpath(cache.hasFastInstanceSize(extraBytes))) { // // 判断是否有缓存
        return cache.fastInstanceSize(extraBytes); // 内存对齐
    }
     // 计算类中所有属性的大小 + 额外的字节数0
    size_t size = alignedInstanceSize() + extraBytes; 
    // 如果size 小于 16，最小取16
    if (size < 16) size = 16;
    return size;
}
```

**fastInstanceSize方法** ：快速计算内存大小方法

```objc
size_t fastInstanceSize(size_t extra) const
{
    ASSERT(hasFastInstanceSize(extra));

    if (__builtin_constant_p(extra) && extra == 0) {
        return _flags & FAST_CACHE_ALLOC_MASK16;
    } else {
        size_t size = _flags & FAST_CACHE_ALLOC_MASK;
        // remove the FAST_CACHE_ALLOC_DELTA16 that was added
        // by setFastInstanceSize
        return align16(size + extra - FAST_CACHE_ALLOC_DELTA16);
    }
}
```

**align16方法** ：内存对齐方法

```objc
static inline size_t align16(size_t x) {
    return (x + size_t(15)) & ~size_t(15);
}

```

**内存对齐算法：**

> 假设传入的参数： x = 8

x + size_t(15) = 8 + 15 = 23 x + size_t(15) 二进制： `0000 0000 0001 0111` = 23 size_t(15) 二进制 ： `0000 0000 0000 1111` = 15 ~size_t(15) : `1111 1111 1111 0000` x + size_t(15) & ~size_t(15)： `0000 0000 0001 0000` = 16 所以返回值 (x + size_t(15)) & ~size_t(15) = 16（原始值：8） 也就是 16 的倍数对齐，即 16 字节对齐

**② calloc：向系统申请开辟内存,返回地址指针** 通过 `instanceSize` 计算的内存大小，向内存中申请大小为 `size` 的内存，并赋值给 `obj` ，因此 `obj` 是指向内存地址的指针

```objc
// alloc 开辟内存的地方
obj = (id)calloc(1, size);
```

这里我们可以通过断点来印证上述的说法，在未执行 `calloc` 时， `po obj` 为 `nil` ，执行后，再 `po obj` 发现，返回了一个16进制的地址 

![[4248d25e1f9c4854924a62d5513dc41b~tplv-k3u1fbpfcp-zoom-1.image.png]]

在平常的开发中，一般一个对象的打印的格式都是类似于这样的<LGPerson: 0x01111111f>（是一个指针）。为什么这里不是呢？

* 主要是因为 `objc` 地址 还没有与传入 的 `cls` 进行 **关联**
* 同时印证了 `alloc` 的根本作用就是 **开辟内存**

**③ obj->initInstanceIsa：关联到相应的类**

![[4b11fe79bc174888bfc172b3dfd7f618~tplv-k3u1fbpfcp-zoom-1.image.png]]


### 7. 总结

根据源码调试方法，我们会得到alloc方法的执行流程如下：

![[deaf81e51b3443dbbbd4f837d79d73e7~tplv-k3u1fbpfcp-zoom-1.image.png]]

# 关于new 和 init

* init方法，如下，可以看到其实init方法什么都没做，只是返回了对象本身
```objc
// Replaced by CF (throws an NSException)
+ (id)init {
    return (id)self;
}
- (id)init {
    return _objc_rootInit(self);
}
id
_objc_rootInit(id obj)
{
    // In practice, it will be hard to rely on this function.
    // Many classes do not properly chain -init calls.
    return obj;
}
```

* new其实最终调用了objc_opt_new，本质上就相当于[[cls alloc] init]
```objc
id objc_opt_new(Class cls)
{
#if __OBJC2__
    if (fastpath(cls && !cls->ISA()->hasCustomCore())) {
        return [callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/) init];
    }
#endif
    return ((id(*)(id, SEL))objc_msgSend)(cls, @selector(new));
}
```

[iOS原理探索系列-alloc&init原理探索](https://juejin.cn/post/6844904098144190471)
[alloc流程分析](https://juejin.cn/post/6953484053693595662)