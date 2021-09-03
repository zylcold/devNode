#### 基础知识
调用时机: [[runtime]] 加载 class 和 category 时调用
调用顺序: 父类、子类、分类
可参考源码 `objc-loadmethod.m`  中

```c++
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}
```

![[Pasted image 9.png]]

#### 调用过程
![[load方法调用过程]]

#### 相关问题
![[category 相关问题#category 中有 load方法 吗]]

![[initialize方法#相关问题]]