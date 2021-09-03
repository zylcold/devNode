1. 通过 runtime 加载某个类所有的 category 数据
2. 把 category 中所有的方法、协议、属性合并到 [[类对象]] 对应的结构中,例如:rw-> method
3. 把 category 中的方法、协议等放在 class 对象中原有方法列表数据的最前面.这就是 category 会覆盖原有方法的本质
见`static void methodizeClass(Class cls)` 方法 和 `attachCategories` 方法, 有说明最后添加的category 在最前面 `oldest categories first.`

```c++
static void methodizeClass(Class cls)
{
    ... ...

    // Install methods and properties that the class implements itself.
    method_list_t *list = ro->baseMethods();
    if (list) {
        prepareMethodLists(cls, &list, 1, YES, isBundleClass(cls));
        rw->methods.attachLists(&list, 1);
    }

    property_list_t *proplist = ro->baseProperties;
    if (proplist) {
        rw->properties.attachLists(&proplist, 1);
    }

    protocol_list_t *protolist = ro->baseProtocols;
    if (protolist) {
        rw->protocols.attachLists(&protolist, 1);
    }

    // Root classes get bonus method implementations if they don't have 
    // them already. These apply before category replacements.
    if (cls->isRootMetaclass()) {
        // root metaclass
        addMethod(cls, SEL_initialize, (IMP)&objc_noop_imp, "", NO);
    }

    // Attach categories.
    category_list *cats = unattachedCategoriesForClass(cls, true /*realizing*/);
    attachCategories(cls, cats, false /*don't flush caches*/);

    ... ...
}
```

```c++
// Attach method lists and properties and protocols from categories to a class.
// Assumes the categories in cats are all loaded and sorted by load order, 
// oldest categories first.
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{

```
 ![[Pasted image 7.png]]
 ![[Pastedimage8.png]]