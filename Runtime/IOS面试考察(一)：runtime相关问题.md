#runtime 

[IOS面试考察(一)：runtime相关问题](https://juejin.cn/post/6844904121531809806 "https://juejin.cn/post/6844904121531809806")

[IOS面试考察(九)：性能优化相关问题](https://juejin.cn/post/6844904131941892110 "https://juejin.cn/post/6844904131941892110")

@[TOC]

# 1. IOS面试考察(一)：runtime相关问题

![[17169dce4b6294bf~tplv-t2oaga2asx-watermark.image.png]]

## 1.1 runtime相关问题

runtime是iOS开发最核心的知识了，如果下面的问题都解决了，那么对runtime的理解已经很深了。 runtime已经开源了，这有一份别人调试好可运行的源码[objc-runtime](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2FRetVal%2Fobjc-runtime "https://github.com/RetVal/objc-runtime")，也可以去官网找[objc4](https://link.juejin.cn/?target=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2Fobjc4%2F "https://opensource.apple.com/tarballs/objc4/")

官方的代码下载下来要让它运行起来，是需要花费一点时间去填坑的。我这里提供一下已经填好坑，可以直接编译运行的objc4_750代码：

-   objc4_750可编译代码：链接:[pan.baidu.com/s/1y0GgOOpF…](https://link.juejin.cn/?target=https%3A%2F%2Fpan.baidu.com%2Fs%2F1y0GgOOpFjUWsmJeVJgtRMg "https://pan.baidu.com/s/1y0GgOOpFjUWsmJeVJgtRMg") 密码:gmlx
-   objc4_756.2可编译代码：链接:[pan.baidu.com/s/15hzKJ12U…](https://link.juejin.cn/?target=https%3A%2F%2Fpan.baidu.com%2Fs%2F15hzKJ12UN4qxZzZPOGYodA "https://pan.baidu.com/s/15hzKJ12UN4qxZzZPOGYodA") 密码:azet

![[17169dce506db4f1~tplv-t2oaga2asx-watermark.image.png]]

先思考一下下面的这些问题，看你能回答出多少：

> 1.  runtime怎么添加属性、方法等
> 2.  runtime 如何实现 weak 属性
> 3.  runtime如何通过selector找到对应的IMP地址？（分别考虑类方法和实例方法）
> 4.  使用runtime Associate方法关联的对象，需要在主对象dealloc的时候释放么？
> 5.  _objc_msgForward函数是做什么的？直接调用它将会发生什么？
> 6.  能否向编译后得到的类中增加实例变量？能否向运行时创建的类中添加实例变量？为什么？
> 7.  简述下Objective-C中调用方法的过程（runtime）
> 8.  什么是method swizzling（俗称黑魔法）

我们先来简单了解一下runtime的3个简单问题：

> 1.  什么是runtime?
> 2.  为啥要用runtime?
> 3.  runtime有什么作用？

-   1.  什么是runtime?

> 1.  runtime本质上是一套比较底层的C语言，C++ ,汇编组成的API。我们有称为运行时，在runtime的底层很多实现为了性能效率方面，都直接用汇编代码。
> 2.  我们平时编写的OC代码，需要runtime来创建类和对象，进行消息发送和转发，其实最终会转换成Runtime的C语言代码。
> 3.  runtime是将数据类型的确定由编译时推迟到了运行时。

-   2.  为啥要用runtime?

> 1.  OC是一门动态语言，它会将一些工作放在代码的运行时才去处理而并非编译时，因此编译器是不够，我们还需要一个运行时系统来处理编译后的代码。
> 2.  runtime基本是用C语言和汇编语言写的，苹果和GNU各自维护一个开源的runtime版本，这两个版本之间都高度的保持一致。

-   3.  runtime有什么作用？

> 1.  消息传递、转发<消息机制>
> 2.  访问私有变量 --eg:(UITextFiled 的修改）
> 3.  交换系统方法 --eg:(拦截--防止button连续点击）
> 4.  动态增加方法
> 5.  为分类增加属性
> 6.  字典转模型 -- eg:(YYModel， MJModel）

接下来我们来给出runtime面试问题的解答，可能不完善，欢迎补充。

### 1.1.1 runtime怎么添加属性、方法等

> ivar表示成员变量 class_addIvar class_addMethod class_addProperty class_addProtocol class_replaceProperty

#### 1.1.1.1 动态添加属性

-   需求:给`NSObject`添加一个name属性,动态添加属性 -> runtime

> 思路：
> 
> 1.  给NSObject添加分类，在分类中添加属性。问题：@property在分类中作用:仅仅是生成get,set方法声明，不会生成get,set方法实现和下划线成员属性，所以要在.m文件实现setter/getter方法，用static保存下滑线属性，这样一来，当对象销毁时，属性无法销毁。
> 2.  用runtime动态添加属性：本质是让属性与某个对象产生一段关联 使用场景:给系统的类添加属性时。

实现代码如下：

```
#import <objc/message.h>
@implementation NSObject (Property)
//static  NSString *_name;      //通过这样去保存属性没法做到对象销毁，属性也销毁，static依然会让属性存在缓存池中，所以需要动态的添加成员属性
// 只要想调用runtime方法,思考:谁的事情
-(void)setName:(NSString *)name
{
    // 保存name
    // 动态添加属性 = 本质:让对象的某个属性与值产生关联
    /*
        object:保存到哪个对象中 
        key:用什么属性保存 属性名
        value:保存值
        policy:策略,strong,weak
     objc_setAssociatedObject(<#id object#>, <#const void *key#>, <#id value#>, <#objc_AssociationPolicy policy#>)
     */
    objc_setAssociatedObject(self, "name", name, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
//    _name = name;

}

- (NSString *)name
{
    return objc_getAssociatedObject(self, "name");
//    return _name;

}
@end
复制代码
```

##### 1.1.1.1.1 自动生成属性

开发中，从网络数据中解析出字典数组，将数组转为模型时，如果有几百个key需要用，要写很多@property成员属性，下面提供一个万能的方法，可直接将字典数组转为全部@property成员属性，打印出来，这样直接复制在模型中就好了。

-   自动生成属性代码如下：

```
#import "NSDictionary+PropertyCode.h"
@implementation NSDictionary (PropertyCode)

//通过这个方法，自动将字典转成模型中需要用的属性代码

// 私有API:真实存在,但是苹果没有暴露出来,不给你。如BOOL值，不知道类型，打印得知是__NSCFBoolean，但是无法敲出来，只能用NSClassFromString(@"__NSCFBoolean")

// isKindOfClass：判断下是否是当前类或者子类，BOOL是NSNumber的子类，要先判断BOOL
- (void)createPropetyCode
{
    // 模型中属性根据字典的key
    // 有多少个key,生成多少个属性
    NSMutableString *codes = [NSMutableString string];
    // 遍历字典
    [self enumerateKeysAndObjectsUsingBlock:^(id  _Nonnull key, id  _Nonnull value, BOOL * _Nonnull stop) {
        NSString *code = nil;
        

//        NSLog(@"%@",[value class]);
        
        if ([value isKindOfClass:[NSString class]]) {
          code = [NSString stringWithFormat:@"@property (nonatomic, strong) NSString *%@;",key];
        } else if ([value isKindOfClass:NSClassFromString(@"__NSCFBoolean")]){
            code = [NSString stringWithFormat:@"@property (nonatomic, assign) BOOL %@;",key];
        } else if ([value isKindOfClass:[NSNumber class]]) {
             code = [NSString stringWithFormat:@"@property (nonatomic, assign) NSInteger %@;",key];
        } else if ([value isKindOfClass:[NSArray class]]) {
            code = [NSString stringWithFormat:@"@property (nonatomic, strong) NSArray *%@;",key];
        } else if ([value isKindOfClass:[NSDictionary class]]) {
            code = [NSString stringWithFormat:@"@property (nonatomic, strong) NSDictionary *%@;",key];
        }
        
        // 拼接字符串
        [codes appendFormat:@"%@\n",code];

    }];
    
    NSLog(@"%@",codes);
    
}

@end
复制代码
```

##### 1.1.1.1.2 KVC字典转模型

-   需求:就是在开发中,通常后台会给你很多数据,但是并不是每个数据都有用,这些没有用的数据,需不需要保存到模型中。 有很多第三方框架都是基于这些原理去实现的，如`YYModel`, `MJModel`.

实现代码如下：

```
@implementation Status
+ (instancetype)statusWithDict:(NSDictionary *)dict{
    // 创建模型
    Status *s = [[self alloc] init];
    
    // 字典value转模型属性保存
    [s setValuesForKeysWithDictionary:dict];

//    s.reposts_count = dict[@"reposts_count"];
    // 4️⃣MJExtension:可以字典转模型,而且可以不用与字典中属性一一对应,runtime,遍历模型中有多少个属性,直接去字典中取出对应value,给模型赋值
    
    // 1️⃣setValuesForKeysWithDictionary:方法底层实现：遍历字典中所有key,去模型中查找对应的属性,把值给模型属性赋值，即调用下面方法：
    /*
    [dict enumerateKeysAndObjectsUsingBlock:^(id  _Nonnull key, id  _Nonnull obj, BOOL * _Nonnull stop) {
        // source
        // 这行代码才是真正给模型的属性赋值
        [s setValue:dict[@"source"] forKey:@"source"];      //底层实现是：
                                 2️⃣ [s setValue:dict[@"source"] forKey:@"source"];
                                 1.首先会去模型中查找有没有setSource方法,直接调用set方法 [s setSource:dict[@"source"]];
                                 2.去模型中查找有没有source属性,source = dict[@"source"]
                                 3.去查找有没有_source属性,_source = dict[@"source"]
                                 4.调用对象的 setValue:forUndefinedKey:直接报错
        [s setValue:obj forKey:key];
    }];
    */
    return s;
}
     

// 3️⃣用KVC，不想让系统报错，重写系统方法思想:
// 1.想给系统方法添加功能
// 2.不想要系统实现
- (void)setValue:(id)value forUndefinedKey:(NSString *)key
{
}


@end
复制代码
```

##### 1.1.1.1.3 MJExtention的底层实现

```
#import "NSObject+Model.h"
#import <objc/message.h>

//    class_copyPropertyList(<#__unsafe_unretained Class cls#>, <#unsigned int *outCount#>) 获取属性列表

@implementation NSObject (Model)

/**
 字典转模型
 @param dict 传入需要转模型的字典
 @return 赋值好的模型
 */

+ (instancetype)modelWithDict:(NSDictionary *)dict

{
    id objc = [[self alloc] init];

    //思路： runtime遍历模型中属性,去字典中取出对应value,在给模型中属性赋值
    // 1.获取模型中所有属性 -> 保存到类
    // ivar:下划线成员变量 和 Property:属性
    // 获取成员变量列表
    // class:获取哪个类成员变量列表
    // count:成员变量总数
    //这个方法得到一个装有成员变量的数组
    //class_copyIvarList(<#__unsafe_unretained Class cls#>, <#unsigned int *outCount#>)
    
    int count = 0;
    // 成员变量数组 指向数组第0个元素
    Ivar *ivarList = class_copyIvarList(self, &count);


    // 遍历所有成员变量
    for (int i = 0; i < count; i++) {
        
        // 获取成员变量 user
        Ivar ivar = ivarList[i];
        // 获取成员变量名称，即将C语言的字符转为OC字符串
        NSString *ivarName = [NSString stringWithUTF8String:ivar_getName(ivar)];
        
        // 获取成员变量类型，用于获取二级字典的模型名字
        NSString *type = [NSString stringWithUTF8String:ivar_getTypeEncoding(ivar)];
        
        //  将type这样的字符串@"@\"User\"" 转成 @"User"
        type = [type stringByReplacingOccurrencesOfString:@"@\"" withString:@""];
        type = [type stringByReplacingOccurrencesOfString:@"\"" withString:@""];
        
        // 成员变量名称转换key，即去掉成员变量前面的下划线
        NSString *key = [ivarName substringFromIndex:1];
        
        // 从字典中取出对应value dict[@"user"] -> 字典
        id value = dict[key];
        
        // 二级转换
        // 并且是自定义类型,才需要转换
        if ([value isKindOfClass:[NSDictionary class]] && ![type containsString:@"NS"]) { // 只有是字典才需要转换
           
            Class className = NSClassFromString(type);
            
            // 字典转模型
            value = [className modelWithDict:value];
        }
        
        // 给模型中属性赋值 key:user value:字典 -> 模型
        if (value) {
            [objc setValue:value forKey:key];
        }
        
    }
    
    return objc;

}

@end
复制代码
```

##### 1.1.1.1.4 自动序列化

-   用runtime提供的函数遍历Model自身所有属性，并对属性进行encode和decode操作。

在Model的基类中重写方法：

```
- (id)initWithCoder:(NSCoder *)aDecoder {
    if (self = [super init]) {
        unsigned int outCount;
        Ivar * ivars = class_copyIvarList([self class], &outCount);
        for (int i = 0; i < outCount; i ++) {
            Ivar ivar = ivars[i];
            NSString * key = [NSString stringWithUTF8String:ivar_getName(ivar)];
            [self setValue:[aDecoder decodeObjectForKey:key] forKey:key];
        }
    }
    return self;
}

- (void)encodeWithCoder:(NSCoder *)aCoder {
    unsigned int outCount;
    Ivar * ivars = class_copyIvarList([self class], &outCount);
    for (int i = 0; i < outCount; i ++) {
        Ivar ivar = ivars[i];
        NSString * key = [NSString stringWithUTF8String:ivar_getName(ivar)];
        [aCoder encodeObject:[self valueForKey:key] forKey:key];
    }
}
复制代码
```

#### 1.1.1.2 动态添加方法

-   开发使用场景：如果一个类方法非常多，加载类到内存的时候也比较耗费资源，需要给每个方法生成映射表，可以使用动态给某个类，添加方法解决。

> 可能的面试题：有没有使用performSelector，其实主要想问你有没有动态添加过方法。

动态添加方法代码实现：

```
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.

    Person *p = [[Person alloc] init];

    // 默认person，没有实现eat方法，可以通过performSelector调用，但是会报错。
    // 动态添加方法就不会报错
    [p performSelector:@selector(eat)];
}

@end


@implementation Person
// void(*)()
// 默认方法都有两个隐式参数，
void eat(id self,SEL sel)
{
    NSLog(@"%@ %@",self,NSStringFromSelector(sel));
}

// 当一个对象调用未实现的方法，会调用这个方法处理,并且会把对应的方法列表传过来.
// 刚好可以用来判断，未实现的方法是不是我们想要动态添加的方法
+ (BOOL)resolveInstanceMethod:(SEL)sel
{
    if (sel == @selector(eat)) {
        // 动态添加eat方法

        // 第一个参数：给哪个类添加方法
        // 第二个参数：添加方法的方法编号
        // 第三个参数：添加方法的函数实现（函数地址）
        // 第四个参数：函数的类型，(返回值+参数类型) v:void @:对象->self :表示SEL->_cmd
        class_addMethod(self, @selector(eat), eat, "v@:");
    }
    return [super resolveInstanceMethod:sel];
}
@end
复制代码
```

#### 1.1.1.3 类,对象的关联对象

OC的分类允许给分类添加属性，但不会自动生成getter、setter方法 所以常规的仅仅添加之后，调用的话会crash。

关联对象不是为类\对象添加属性或者成员变量(因为在设置关联后也无法通过ivarList或者propertyList取得) ，而是为类添加一个相关的对象，通常用于存储类信息，例如存储类的属性列表数组，为将来字典转模型的方便。

runtime关联对象的API：

```
//关联对象
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
//获取关联的对象
id objc_getAssociatedObject(id object, const void *key)
//移除关联的对象
void objc_removeAssociatedObjects(id object)
复制代码
```

-   给分类添加属性：

```
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // 给系统NSObject类动态添加属性name
    NSObject *objc = [[NSObject alloc] init];
    objc.name = @"孔雨露";
    NSLog(@"%@",objc.name);
}
@end


// 定义关联的key
static const char *key = "name";

@implementation NSObject (Property)

- (NSString *)name
{
    // 根据关联的key，获取关联的值。
    return objc_getAssociatedObject(self, key);
}

- (void)setName:(NSString *)name
{
    // 第一个参数：给哪个对象添加关联
    // 第二个参数：关联的key，通过这个key获取
    // 第三个参数：关联的value
    // 第四个参数:关联的策略
    objc_setAssociatedObject(self, key, name, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

@end
复制代码
```

-   给对象添加关联对象： 比如alertView,一般传值，使用的是alertView的tag属性。我们想把更多的参数传给alertView代理：

```
/**
 *  删除点击
 *  @param recId        购物车ID
 */
- (void)shopCartCell:(BSShopCartCell *)shopCartCell didDeleteClickedAtRecId:(NSString *)recId
{
    UIAlertView *alert = [[UIAlertView alloc] initWithTitle:@"" message:@"确认要删除这个宝贝" delegate:self cancelButtonTitle:@"取消" otherButtonTitles:@"确定", nil];
    /*
    objc_setAssociatedObject方法的参数解释:
    第一个参数: id object, 当前对象
    第二个参数: const void *key, 关联的key，是c字符串
	第三个参数: id value, 被关联的对象的值
	第四个参数: objc_AssociationPolicy policy关联引用的规则
   */
    // 传递多参数
    objc_setAssociatedObject(alert, "suppliers_id", @"1", OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    objc_setAssociatedObject(alert, "warehouse_id", @"2", OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    
    alert.tag = [recId intValue];
    [alert show];
}

/**
 *  确定删除操作
 */
- (void)alertView:(UIAlertView *)alertView clickedButtonAtIndex:(NSInteger)buttonIndex {
    if (buttonIndex == 1) {
        
        NSString *warehouse_id = objc_getAssociatedObject(alertView, "warehouse_id");
        NSString *suppliers_id = objc_getAssociatedObject(alertView, "suppliers_id");
        NSString *recId = [NSString stringWithFormat:@"%ld",(long)alertView.tag];
    }
}
复制代码
```

### 1.1.2 runtime 如何实现 weak 属性

-   首先要搞清楚weak属性的特点:

> weak策略表明该属性定义了一种“非拥有关系” (nonowning relationship)。为这种属性设置新值时，设置方法既不保留新值，也不释放旧值。此特质同assign类似;然而在属性所指的对象遭到摧毁时，属性值也会清空(nil out)

-   那么runtime如何实现weak变量的自动置nil？

> runtime对注册的类，会进行布局，会将 weak 对象放入一个 hash 表中。用 weak 指向的对象内存地址作为 key，当此对象的引用计数为0的时候会调用对象的 dealloc 方法，假设 weak 指向的对象内存地址是a，那么就会以a为key，在这个 weak hash表中搜索，找到所有以a为key的 weak 对象，从而设置为 nil。

-   weak属性需要在dealloc中置nil么?

> 在ARC环境无论是强指针还是弱指针都无需在 dealloc 设置为 nil ， ARC 会自动帮我们处理 即便是编译器不帮我们做这些，weak也不需要在dealloc中置nil 在属性所指的对象遭到摧毁时，属性值也会清空

```
objc// 模拟下weak的setter方法，大致如下
- (void)setObject:(NSObject *)object{   

  objc_setAssociatedObject(self, "object", object, OBJC_ASSOCIATION_ASSIGN);

 [object cyl_runAtDealloc:^{ _object = nil; }];
 }

复制代码
```

先看下 runtime 里源码的实现：具体完整实现参照 [objc/objc-weak.h](https://link.juejin.cn/?target=https%3A%2F%2Fopensource.apple.com%2Fsource%2Fobjc4%2Fobjc4-646%2Fruntime%2Fobjc-weak.h "https://opensource.apple.com/source/objc4/objc4-646/runtime/objc-weak.h") 。

```
/**
* The internal structure stored in the weak references table. 
* It maintains and stores
* a hash set of weak references pointing to an object.
* If out_of_line==0, the set is instead a small inline array.
*/
#define WEAK_INLINE_COUNT 4
struct weak_entry_t {
   DisguisedPtr<objc_object> referent;
   union {
       struct {
           weak_referrer_t *referrers;
           uintptr_t        out_of_line : 1;
           uintptr_t        num_refs : PTR_MINUS_1;
           uintptr_t        mask;
           uintptr_t        max_hash_displacement;
       };
       struct {
           // out_of_line=0 is LSB of one of these (don't care which)
           weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
       };
   };
};

/**
* The global weak references table. Stores object ids as keys,
* and weak_entry_t structs as their values.
*/
struct weak_table_t {
   weak_entry_t *weak_entries;
   size_t    num_entries;
   uintptr_t mask;
   uintptr_t max_hash_displacement;
};
复制代码
```

我们可以设计一个函数（伪代码）来表示上述机制：

`objc_storeWeak(&a, b)`函数：

> `objc_storeWeak`函数把第二个参数--赋值对象（b）的内存地址作为键值key，将第一个参数--`weak`修饰的属性变量（a）的内存地址（&a）作为`value`，注册到 `weak` 表中。如果第二个参数（b）为0（nil），那么把变量（a）的内存地址（&a）从`weak`表中删除，

你可以把`objc_storeWeak(&a, b)`理解为：`objc_storeWeak(value, key)`，并且当`key`变`nil`，将`value`置`nil`。

在b非nil时，a和b指向同一个内存地址，在b变nil时，a变nil。此时向a发送消息不会崩溃：在Objective-C中向nil发送消息是安全的。

而如果a是由 assign 修饰的，则： 在 b 非 nil 时，a 和 b 指向同一个内存地址，在 b 变 nil 时，a 还是指向该内存地址，变野指针。此时向 a 发送消息极易崩溃。

下面我们将基于`objc_storeWeak(&a, b)`函数，使用伪代码模拟“runtime如何实现weak属性”：

```
// 使用伪代码模拟：runtime如何实现weak属性
 id obj1;
 objc_initWeak(&obj1, obj);
/*obj引用计数变为0，变量作用域结束*/
 objc_destroyWeak(&obj1);
复制代码
```

下面对用到的两个方法`objc_initWeak`和`objc_destroyWeak`做下解释：

> 总体说来，作用是： 通过`objc_initWeak`函数初始化“附有`weak`修饰符的变量（obj1）”，在变量作用域结束时通过`objc_destoryWeak`函数释放该变量（obj1）。

下面分别介绍下方法的内部实现：

`objc_initWeak`函数的实现是这样的：在将“附有`weak`修饰符的变量（obj1）”初始化为0（nil）后，会将“赋值对象”（obj）作为参数，调用`objc_storeWeak`函数。

```
obj1 = 0；
obj_storeWeak(&obj1, obj);
复制代码
```

也就是说：`weak` 修饰的指针默认值是 nil （在Objective-C中向nil发送消息是安全的）。 然后`obj_destroyWeak`函数将0（nil）作为参数，调用`objc_storeWeak`函数。`objc_storeWeak(&obj1, 0);` 前面的源代码与下列源代码相同。

```
// 使用伪代码模拟：runtime如何实现weak属性
id obj1;
obj1 = 0;
objc_storeWeak(&obj1, obj);
/* ... obj的引用计数变为0，被置nil ... */
objc_storeWeak(&obj1, 0);
复制代码
```

`objc_storeWeak` 函数把第二个参数--赋值对象（obj）的内存地址作为键值，将第一个参数--weak修饰的属性变量（obj1）的内存地址注册到 weak 表中。如果第二个参数（obj）为0（nil），那么把变量（obj1）的地址从 weak 表中删除，

使用伪代码是为了方便理解，下面我们“真枪实弹”地实现下：

-   如何让不使用weak修饰的@property，拥有weak的效果。

我们从setter方法入手：

（注意以下的 `cyl_runAtDealloc` 方法实现仅仅用于模拟原理，如果想用于项目中，还需要考虑更复杂的场景，想在实际项目使用的话，可以使用ChenYilong 写的一个库 [CYLDeallocBlockExecutor](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2FChenYilong%2FCYLDeallocBlockExecutor "https://github.com/ChenYilong/CYLDeallocBlockExecutor") ）

```
- (void)setObject:(NSObject *)object
{
   objc_setAssociatedObject(self, "object", object, OBJC_ASSOCIATION_ASSIGN);
   [object cyl_runAtDealloc:^{
       _object = nil;
   }];
}
复制代码
```

也就是有两个步骤：

(1) 在setter方法中做如下设置：

```
objc_setAssociatedObject(self, "object", object, OBJC_ASSOCIATION_ASSIGN);

复制代码
```

(2), 在属性所指的对象遭到摧毁时，属性值也会清空(nil out)。做到这点，同样要借助 runtime：

```
//要销毁的目标对象
id objectToBeDeallocated;
//可以理解为一个“事件”：当上面的目标对象销毁时，同时要发生的“事件”。
id objectWeWantToBeReleasedWhenThatHappens;
objc_setAssociatedObject(objectToBeDeallocted,
                        someUniqueKey,
                        objectWeWantToBeReleasedWhenThatHappens,
                        OBJC_ASSOCIATION_RETAIN);
复制代码
```

知道了思路，我们就开始实现 cyl_runAtDealloc 方法，实现过程分两部分：

-   第一部分：创建一个类，可以理解为一个“事件”：当目标对象销毁时，同时要发生的“事件”。借助 block 执行“事件”。

```
// 这个类，可以理解为一个“事件”：当目标对象销毁时，同时要发生的“事件”。借助block执行“事件”。

typedef void (^voidBlock)(void);

@interface CYLBlockExecutor : NSObject
- (id)initWithBlock:(voidBlock)block;
@end

@interface CYLBlockExecutor() {
   voidBlock _block;
}
@implementation CYLBlockExecutor

- (id)initWithBlock:(voidBlock)aBlock
{
   self = [super init];
   if (self) {
       _block = [aBlock copy];
   }
   return self;
}

- (void)dealloc
{
   _block ? _block() : nil;
}

@end
复制代码
```

-   第二部分：核心代码：利用runtime实现cyl_runAtDealloc方法

```
// 利用runtime实现cyl_runAtDealloc方法

#import "CYLBlockExecutor.h"

const void *runAtDeallocBlockKey = &runAtDeallocBlockKey;

@interface NSObject (CYLRunAtDealloc)
- (void)cyl_runAtDealloc:(voidBlock)block;
@end

@implementation NSObject (CYLRunAtDealloc)

- (void)cyl_runAtDealloc:(voidBlock)block
{
   if (block) {
       CYLBlockExecutor *executor = [[CYLBlockExecutor alloc] initWithBlock:block];
       
       objc_setAssociatedObject(self,
                                runAtDeallocBlockKey,
                                executor,
                                OBJC_ASSOCIATION_RETAIN);
   }
}

@end
复制代码
```

使用方法： 导入`#import "CYLNSObject+RunAtDealloc.h"`,然后就可以使用了：

```
NSObject *foo = [[NSObject alloc] init];

[foo cyl_runAtDealloc:^{
   NSLog(@"正在释放foo!");
}];
复制代码
```

#### 1.1.2.1 runtime如何实现weak变量的自动置nil？

> `runtime` 对注册的类， 会进行布局，对于 `weak` 对象会放入一个 `hash` 表中。 用 `weak` 指向的对象内存地址作为 `key`，当此对象的引用计数为0的时候会 `dealloc`，假如 `weak` 指向的对象内存地址是a，那么就会以a为键， 在这个 `weak` 表中搜索，找到所有以a为键的 `weak` 对象，从而设置为 nil。

我们可以设计一个函数（伪代码）来表示上述机制：

`objc_storeWeak(&a, b)`函数：

-   `objc_storeWeak`函数把第二个参数--赋值对象（b）的内存地址作为键值key，将第一个参数--`weak`修饰的属性变量（a）的内存地址（&a）作为value，注册到 `weak` 表中。如果第二个参数（b）为0（nil），那么把变量（a）的内存地址（&a）从`weak`表中删除，
    
-   你可以把`objc_storeWeak(&a, b`)理解为：`objc_storeWeak(value, key)`，并且当`key`变`nil`，将`value`置`nil`。
    
-   在b非`nil`时，a和b指向同一个内存地址，在b变nil时，a变nil。此时向a发送消息不会崩溃：在Objective-C中向nil发送消息是安全的。
    
-   而如果a是由`assign`修饰的，则： 在b非nil时，a和b指向同一个内存地址，在b变nil时，a还是指向该内存地址，变野指针。此时向a发送消息极易崩溃。
    

下面我们将基于`objc_storeWeak(&a, b)`函数，使用伪代码模拟“runtime如何实现weak属性”：

```
 id obj1;
 objc_initWeak(&obj1, obj);
/*obj引用计数变为0，变量作用域结束*/
 objc_destroyWeak(&obj1);
复制代码
```

下面对用到的两个方法`objc_initWeak`和`objc_destroyWeak`做下解释：

总体说来，作用是： 通过`objc_initWeak`函数初始化“附有weak修饰符的变量（obj1）”，在变量作用域结束时通过`objc_destoryWeak`函数释放该变量（obj1）。

下面分别介绍下方法的内部实现：

`objc_initWeak`函数的实现是这样的：在将“附有`weak`修饰符的变量（obj1）”初始化为0（nil）后，会将“赋值对象”（obj）作为参数，调用`objc_storeWeak`函数。

```
obj1 = 0；
obj_storeWeak(&obj1, obj);
复制代码
```

也就是说：weak 修饰的指针默认值是 nil （在Objective-C中向nil发送消息是安全的） 然后obj_destroyWeak函数将0（nil）作为参数，调用`objc_storeWeak`函数。

```
objc_storeWeak(&obj1, 0);
复制代码
```

```
id obj1;
obj1 = 0;
objc_storeWeak(&obj1, obj);
/* ... obj的引用计数变为0，被置nil ... */
objc_storeWeak(&obj1, 0);
复制代码
```

`objc_storeWeak`函数把第二个参数--赋值对象（obj）的内存地址作为键值，将第一个参数--weak修饰的属性变量（obj1）的内存地址注册到 `weak` 表中。如果第二个参数（obj）为0（nil），那么把变量（obj1）的地址从`weak`表中删除。

### 1.1.3 runtime如何通过selector找到对应的IMP地址？（分别考虑类方法和实例方法）

> 1.  每一个类对象中都一个对象方法列表（对象方法缓存）
> 2.  类方法列表是存放在类对象中isa指针指向的元类对象中（类方法缓存）
> 3.  方法列表中每个方法结构体中记录着方法的名称,方法实现,以及参数类型，其实selector本质就是方法名称,通过这个方法名称就可以在方法列表中找到对应的方法实现.
> 4.  当我们发送一个消息给一个NSObject对象时，这条消息会在对象的类对象方法列表里查找
> 5.  当我们发送一个消息给一个类时，这条消息会在类的Meta Class对象的方法列表里查找

> **元类**，就像之前的类一样，它也是一个对象，所有的元类都使用根元类（继承体系中处于顶端的类的元类）作为他们的类。这就意味着所有NSObject的子类（大多数类）的元类都会以NSObject的元类作为他们的类，根据这个规则，所有的元类使用根元类作为他们的类，根元类的元类则就是它自己。也就是说基类的元类的isa指针指向他自己。

### 1.1.4 使用runtime Associate方法关联的对象，需要在主对象dealloc的时候释放么？

无论在MRC下还是ARC下均不需要, 被关联的对象在生命周期内要比对象本身释放的晚很多，它们会在被 NSObject -dealloc 调用的object_dispose()方法中释放 补充：对象的内存销毁时间表，分四个步骤

> 1.  调用 -release ：引用计数变为零 对象正在被销毁，生命周期即将结束. 不能再有新的 __weak 弱引用，否则将指向 nil. 调用 [self dealloc]
> 2.  父类调用 -dealloc 继承关系中最直接继承的父类再调用 -dealloc 如果是 MRC 代码 则会手动释放实例变量们（iVars） 继承关系中每一层的父类 都再调用 -dealloc
> 3.  NSObject 调 -dealloc 只做一件事：调用 Objective-C runtime 中object_dispose() 方法
> 4.  调用 object_dispose() 为 C++ 的实例变量们（iVars）调用 destructors 为 ARC 状态下的 实例变量们（iVars） 调用 -release 解除所有使用 runtime Associate方法关联的对象 解除所有 __weak 引用 调用 free()

### 1.1.5 _objc_msgForward函数是做什么的？直接调用它将会发生什么？

_objc_msgForward是IMP类型，用于消息转发的：当向一个对象发送一条消息，但它并没有实现的时候，_objc_msgForward会尝试做消息转发。 直接调用_objc_msgForward是非常危险 的事，这是把双刃刀，如果用不好会直接导致程序Crash，但是如果用得好，能做很多非常酷的事 [JSPatch](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fbang590%2FJSPatch "https://github.com/bang590/JSPatch")就是直接调用_objc_msgForward来实现其核心功能的 [详细解说参见这里的第一个问题解答](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2FChenYilong%2FiOSInterviewQuestions%2Fblob%2Fmaster%2F01%25E3%2580%258A%25E6%258B%259B%25E8%2581%2598%25E4%25B8%2580%25E4%25B8%25AA%25E9%259D%25A0%25E8%25B0%25B1%25E7%259A%2584iOS%25E3%2580%258B%25E9%259D%25A2%25E8%25AF%2595%25E9%25A2%2598%25E5%258F%2582%25E8%2580%2583%25E7%25AD%2594%25E6%25A1%2588%2F%25E3%2580%258A%25E6%258B%259B%25E8%2581%2598%25E4%25B8%2580%25E4%25B8%25AA%25E9%259D%25A0%25E8%25B0%25B1%25E7%259A%2584iOS%25E3%2580%258B%25E9%259D%25A2%25E8%25AF%2595%25E9%25A2%2598%25E5%258F%2582%25E8%2580%2583%25E7%25AD%2594%25E6%25A1%2588%25EF%25BC%2588%25E4%25B8%258B%25EF%25BC%2589.md "https://github.com/ChenYilong/iOSInterviewQuestions/blob/master/01%E3%80%8A%E6%8B%9B%E8%81%98%E4%B8%80%E4%B8%AA%E9%9D%A0%E8%B0%B1%E7%9A%84iOS%E3%80%8B%E9%9D%A2%E8%AF%95%E9%A2%98%E5%8F%82%E8%80%83%E7%AD%94%E6%A1%88/%E3%80%8A%E6%8B%9B%E8%81%98%E4%B8%80%E4%B8%AA%E9%9D%A0%E8%B0%B1%E7%9A%84iOS%E3%80%8B%E9%9D%A2%E8%AF%95%E9%A2%98%E5%8F%82%E8%80%83%E7%AD%94%E6%A1%88%EF%BC%88%E4%B8%8B%EF%BC%89.md")

我们可以这样创建一个_objc_msgForward对象：`IMP msgForwardIMP = _objc_msgForward;`。

我们知道objc_msgSend在“消息传递”中的作用。 在“消息传递”过程中，objc_msgSend的动作比较清晰：

> 1.  首先在 Class 中的缓存查找 IMP （没缓存则初始化缓存），
> 2.  如果没找到，则向父类的 Class 查找。
> 3.  如果一直查找到根类仍旧没有实现，则用_objc_msgForward函数指针代替 IMP 。
> 4.  最后，执行这个 IMP 。

我们可以在objc4_750源码中的`objc-runtime-new.mm`文件中搜索`_objc_msgForward`里面有一个`lookUpImpOrForward`的说明

```
/***********************************************************************
* lookUpImpOrForward.
* The standard IMP lookup. 
* initialize==NO tries to avoid +initialize (but sometimes fails)
* cache==NO skips optimistic unlocked lookup (but uses cache elsewhere)
* Most callers should use initialize==YES and cache==YES.
* inst is an instance of cls or a subclass thereof, or nil if none is known. 
*   If cls is an un-initialized metaclass then a non-nil inst is faster.
* May return _objc_msgForward_impcache. IMPs destined for external use 
*   must be converted to _objc_msgForward or _objc_msgForward_stret.
*   If you don't want forwarding at all, use lookUpImpOrNil() instead.
**********************************************************************/
复制代码
```

对 `objc-runtime-new.mm`文件里与`_objc_msgForward`有关的三个函数使用伪代码展示下：

```
id objc_msgSend(id self, SEL op, ...) {
    if (!self) return nil;
	IMP imp = class_getMethodImplementation(self->isa, SEL op);
	imp(self, op, ...); //调用这个函数，伪代码...
}
 
//查找IMP
IMP class_getMethodImplementation(Class cls, SEL sel) {
    if (!cls || !sel) return nil;
    IMP imp = lookUpImpOrNil(cls, sel);
    if (!imp) return _objc_msgForward; //_objc_msgForward 用于消息转发
    return imp;
}
 
IMP lookUpImpOrNil(Class cls, SEL sel) {
    if (!cls->initialize()) {
        _class_initialize(cls);
    }
 
    Class curClass = cls;
    IMP imp = nil;
    do { //先查缓存,缓存没有时重建,仍旧没有则向父类查询
        if (!curClass) break;
        if (!curClass->cache) fill_cache(cls, curClass);
        imp = cache_getImp(curClass, sel);
        if (imp) break;
    } while (curClass = curClass->superclass);
 
    return imp;
}
复制代码
```

虽然Apple没有公开_objc_msgForward的实现源码，但是我们还是能得出结论：

> 1.  `_objc_msgForward`是一个函数指针（和 `IMP` 的类型一样），是用于消息转发的：当向一个对象发送一条消息，但它并没有实现的时候，`_objc_msgForward`会尝试做消息转发。
> 2.  `objc_msgSend`在“消息传递”中的作用: 在“消息传递”过程中，`objc_msgSend`的动作比较清晰： (1) 首先在 `Class` 中的缓存查找 `IMP` （没缓存则初始化缓存）， (2) 如果没找到，则向父类的 `Class`查找。 (3) 如果一直查找到根类仍旧没有实现，则用`_objc_msgForward`函数指针代替 `IMP` 。 (4) 最后，执行这个 `IMP` 。

为了展示消息转发的具体动作，这里尝试向一个对象发送一条错误的消息，并查看一下`_objc_msgForward`是如何进行转发的。

首先开启调试模式、打印出所有运行时发送的消息： 可以在代码里执行下面的方法：`(void)instrumentObjcMessageSends(YES);`

因为该函数处于 objc-internal.h 内，而该文件并不开放，所以调用的时候先声明，目的是告诉编译器程序目标文件包含该方法存在，让编译通过:

```
OBJC_EXPORT void
instrumentObjcMessageSends(BOOL flag)
OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0, 2.0);
复制代码
```

或者断点暂停程序运行，并在 gdb 中输入下面的命令：`call (void)instrumentObjcMessageSends(YES)`

以第二种为例，操作如下所示：

![[17169dce8532298b~tplv-t2oaga2asx-watermark.image.png]]

之后，运行时发送的所有消息都会打印到/tmp/msgSend-xxxx文件里了。

终端中输入命令前往：`open /private/tmp`

![[17169dce48f6fb35~tplv-t2oaga2asx-watermark.image.png]]

可能看到有多条，找到最新生成的，双击打开

在模拟器上执行执行以下语句（这一套调试方案仅适用于模拟器，真机不可用，关于该调试方案的拓展链接： [Can the messages sent to an object in Objective-C be monitored or printed out? ）](https://link.juejin.cn/?target=https%3A%2F%2Fstackoverflow.com%2Fquestions%2F10749452%2Fcan-the-messages-sent-to-an-object-in-objective-c-be-monitored-or-printed-out%2F10750398%2310750398 "https://stackoverflow.com/questions/10749452/can-the-messages-sent-to-an-object-in-objective-c-be-monitored-or-printed-out/10750398#10750398")，向一个对象发送一条错误的消息：

![[17169dce48db1798~tplv-t2oaga2asx-watermark.image.png]]

你可以在/tmp/msgSend-xxxx（我这一次是/tmp/msgSend-9805）文件里，看到打印出来：

![[17169dce53050452~tplv-t2oaga2asx-watermark.image.png]]

结合[《NSObject官方文档》](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.apple.com%2Fdocumentation%2Fobjectivec%2Fnsobject%23%2F%2Fapple_ref%2Fdoc%2Fuid%2F20000050-SW11 "https://developer.apple.com/documentation/objectivec/nsobject#//apple_ref/doc/uid/20000050-SW11")，排除掉 NSObject 做的事，剩下的就是_objc_msgForward消息转发做的几件事：

> 1.  调用`resolveInstanceMethod:`方法 (或 `resolveClassMethod`:)。允许用户在此时为该 `Class` 动态添加实现。如果有实现了，则调用并返回`YES`，那么重新开始`objc_msgSend`流程。这一次对象会响应这个选择器，一般是因为它已经调用过`class_addMethod`。如果仍没实现，继续下面的动作。
> 2.  调用`forwardingTargetForSelector`:方法，尝试找到一个能响应该消息的对象。如果获取到，则直接把消息转发给它，返回非 nil 对象。否则返回 nil ，继续下面的动作。注意，这里不要返回 `self` ，否则会形成死循环。
> 3.  调用`methodSignatureForSelector`:方法，尝试获得一个方法签名。如果获取不到，则直接调用`doesNotRecognizeSelector`抛出异常。如果能获取，则返回非`nil`：创建一个 `NSlnvocation`并传给`forwardInvocation`:。
> 4.  调用`forwardInvocation`:方法，将第3步获取到的方法签名包装成 `Invocation` 传入，如何处理就在这里面了，并返回非nil。
> 5.  调用`doesNotRecognizeSelector`: ，默认的实现是抛出异常。如果第3步没能获得一个方法签名，执行该步骤。

上面前4个方法均是模板方法，开发者可以`override`，由 `runtime` 来调用。最常见的实现消息转发：就是重写方法3和4，吞掉一个消息或者代理给其他对象都是没问题的。 也就是说_objc_msgForward在进行消息转发的过程中会涉及以下这几个方法：

> 1.  resolveInstanceMethod:方法 (或 resolveClassMethod:)。
> 2.  forwardingTargetForSelector:方法
> 3.  methodSignatureForSelector:方法
> 4.  forwardInvocation:方法
> 5.  doesNotRecognizeSelector: 方法
>     
>     ![[17169dce4d78dac3~tplv-t2oaga2asx-watermark.image.png]]
>     

-   下面回答下第二个问题“直接_objc_msgForward调用它将会发生什么？”

直接调用_objc_msgForward是非常危险的事，如果用不好会直接导致程序Crash，但是如果用得好，能做很多非常酷的事。

就好像跑酷，干得好，叫“耍酷”，干不好就叫“作死”。

正如前文所说：`_objc_msgForward`是 `IMP` 类型，用于消息转发的：当向一个对象发送一条消息，但它并没有实现的时候，`_objc_msgForward`会尝试做消息转发。

如何调用_objc_msgForward？ _objc_msgForward隶属 C 语言，有三个参数 ：

-- | -- | --
--  |  _objc_msgForward参数  | 类型
1  | 所属对象 |   id类型
2 | 方法名 | SEL类型
3 | 可变参数 | 可变参数类型


首先了解下如何调用 IMP 类型的方法，IMP类型是如下格式：

为了直观，我们可以通过如下方式定义一个 IMP类型 ：

```
typedef void (*voidIMP)(id, SEL, ...)
复制代码
```

一旦调用`_objc_msgForward`，将跳过查找 `IMP` 的过程，直接触发“消息转发”，

如果调用了`_objc_msgForward`，即使这个对象确实已经实现了这个方法，你也会告诉`objc_msgSend`：“我没有在这个对象里找到这个方法的实现”

想象下`objc_msgSend`会怎么做？通常情况下，下面这张图就是你正常走`objc_msgSend`过程，和直接调用`_objc_msgForward`的前后差别：

![[17169dce8f3105e1~tplv-t2oaga2asx-watermark.image.gif]]

### 1.1.6 能否向编译后得到的类中增加实例变量？能否向运行时创建的类中添加实例变量？为什么？

不能向编译后得到的类中增加实例变量；能向运行时创建的类中添加实例变量；

> 因为编译后的类已经注册在runtime中，类结构体中的objc_ivar_list 实例变量的链表和instance_size实例变量的内存大小已经确定，同时runtime 会调用class_setIvarLayout 或 class_setWeakIvarLayout来处理strong weak引用，所以不能向存在的类中添加实例变量 运行时创建的类是可以添加实例变量，调用 class_addIvar函数，但是得在调用objc_allocateClassPair之后，objc_registerClassPair之前，原因同上。

### 1.1.7 简述下Objective-C中调用方法的过程（runtime）

Runtime 铸就了Objective-C 是动态语言的特性，使得C语言具备了面向对象的特性，在程序运行期创建，检查，修改类、对象及其对应的方法，这些操作都可以使用runtime中的对应方法实现。

Objective-C是动态语言，每个方法在运行时会被动态转为消息发送，即：objc_msgSend(receiver, selector)，整个过程介绍如下：

> 1.  objc在向一个对象发送消息时，runtime库会根据对象的isa指针找到该对象实际所属的类
> 2.  然后在该类中的方法列表以及其父类方法列表中寻找方法运行
> 3.  如果，在最顶层的父类（一般也就NSObject）中依然找不到相应的方法时，程序在运行时会挂掉并抛出异常unrecognized selector sent to XXX
> 4.  但是在这之前，objc的运行时会给出三次拯救程序崩溃的机会。这三次机会分别是： （1）动态方法解析过程中的：对象方法动态解析（`+(BOOL)resolveInstanceMethod:(SEL)sel`）和 类方法动态解析 （`+(BOOL)resolveClassMethod:(SEL)sel`） （2）如果动态解析失败，则会进入消息转发流程，消息转发又分为：快速转发和慢速转发两种方式。 （3）快速转发的实现是 `forwardingTargetForSelector`，让其他能响应要查找消息的对象来干活。 （4）慢速转发的实现是 `methodSignatureForSelector` 和 `forwardInvocation` 的结合，提供了更细粒度的控制，先返回方法签名给 `Runtime`，然后让 `anInvocation` 来把消息发送给提供的对象，最后由 `Runtime`提取结果然后传递给原始的消息发送者。 （5）如果在3次挽救机会：`resolveInstanceMethod`，`forwardingTargetForSelector`，`forwardInvocation`都没有处理时，就会报`unrecognized selector sent to XXX`异常。此时程序会崩溃。

![[17169dce4d78dac3~tplv-t2oaga2asx-watermark.image 1.png]]

-   关于`resolveInstanceMethod` 方法又称为对象方法动态解析,它的流程大致如下：

> 1.  检查是否实现了 `+(BOOL)resolveInstanceMethod:(SEL)sel` 类方法，如果没有实现则直接返回(通过 `cls->ISA()` 是拿到元类，因为类方法是存储在元类上的对象方法)
> 2.  如果当前实现了 `+(BOOL)resolveInstanceMethod:(SEL)sel` 类方法，则通过 `objc_msgSend` 手动调用该类方法。
> 3.  完成调用后，再次查询 `cls` 中的 `imp`。
> 4.  如果 `imp` 找到了，则输出动态解析对象方法成功的日志。
> 5.  如果 `imp` 没有找到，则输出虽然实现了 `+(BOOL)resolveInstanceMethod:(SEL)sel`，并且返回了 `YES`。

对应源码如下：

```
static void _class_resolveInstanceMethod(Class cls, SEL sel, id inst)
{
    if (! lookUpImpOrNil(cls->ISA(), SEL_resolveInstanceMethod, cls, 
                         NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
    {
        // Resolver not implemented.
        return;
    }

    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
    bool resolved = msg(cls, SEL_resolveInstanceMethod, sel);

    // Cache the result (good or bad) so the resolver doesn't fire next time.
    // +resolveInstanceMethod adds to self a.k.a. cls
    IMP imp = lookUpImpOrNil(cls, sel, inst, 
                             NO/*initialize*/, YES/*cache*/, NO/*resolver*/);

    if (resolved  &&  PrintResolving) {
        if (imp) {
            _objc_inform("RESOLVE: method %c[%s %s] "
                         "dynamically resolved to %p", 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel), imp);
        }
        else {
            // Method resolver didn't add anything?
            _objc_inform("RESOLVE: +[%s resolveInstanceMethod:%s] returned YES"
                         ", but no new implementation of %c[%s %s] was found",
                         cls->nameForLogging(), sel_getName(sel), 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel));
        }
    }
}
复制代码
```

-   如果对象方法动态解析未实现，实际会沿着isa指针，去调用`_class_resolveClassMethod` 走类方法动态解析流程：

> 1.  判断是否是元类，如果不是，直接退出。
> 2.  检查是否实现了 `+(BOOL)resolveClassMethod:(SEL)sel` 类方法，如果没有实现则直接返回(通过 `cls-` 是因为当前 `cls` 就是元类，因为类方法是存储在元类上的对象方法)
> 3.  如果当前实现了 `+(BOOL)resolveClassMethod:(SEL)sel` 类方法，则通过 `objc_msgSend` 手动调用该类方法，注意这里和动态解析对象方法不同，这里需要通过元类和对象来找到类，也就是 `_class_getNonMetaClass`。
> 4.  完成调用后，再次查询 `cls` 中的 `imp`。
> 5.  如果 `imp` 找到了，则输出动态解析对象方法成功的日志。
> 6.  如果 `imp` 没有找到，则输出虽然实现了 `+(BOOL)resolveClassMethod:(SEL)sel`，并且返回了 `YES`，但并没有查找到 `imp` 的日志

对应源码如下：

```
static void _class_resolveClassMethod(Class cls, SEL sel, id inst)
{
    assert(cls->isMetaClass());

    if (! lookUpImpOrNil(cls, SEL_resolveClassMethod, inst, 
                         NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
    {
        // Resolver not implemented.
        return;
    }

    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
    bool resolved = msg(_class_getNonMetaClass(cls, inst), 
                        SEL_resolveClassMethod, sel);

    // Cache the result (good or bad) so the resolver doesn't fire next time.
    // +resolveClassMethod adds to self->ISA() a.k.a. cls
    IMP imp = lookUpImpOrNil(cls, sel, inst, 
                             NO/*initialize*/, YES/*cache*/, NO/*resolver*/);

    if (resolved  &&  PrintResolving) {
        if (imp) {
            _objc_inform("RESOLVE: method %c[%s %s] "
                         "dynamically resolved to %p", 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel), imp);
        }
        else {
            // Method resolver didn't add anything?
            _objc_inform("RESOLVE: +[%s resolveClassMethod:%s] returned YES"
                         ", but no new implementation of %c[%s %s] was found",
                         cls->nameForLogging(), sel_getName(sel), 
                         cls->isMetaClass() ? '+' : '-', 
                         cls->nameForLogging(), sel_getName(sel));
        }
    }
}
复制代码
```

-   关于`forwardingTargetForSelector`方法 又称为快速消息转发：

> 1.  `forwardingTargetForSelector` 是一种快速的消息转发流程，它直接让其他对象来响应未知的消息。
> 2.  `forwardingTargetForSelector` 不能返回 `self`，否则会陷入死循环，因为返回 `self` 又回去当前实例对象身上走一遍消息查找流程，显然又会来到 `forwardingTargetForSelector`。
> 3.  `forwardingTargetForSelector` 适用于消息转发给其他能响应未知消息的对象，也就是最终返回的内容必须和要查找的消息的参数和返回值一致，如果想要不一致，就需要走其他的流程。

快速消息转发是通过汇编来实现的，根据 `lookUpImpOrForward` 源码我们可以看到当动态解析没有成功后，会直接返回一个 `_objc_msgForward_impcache`。 在源码中全局搜索`_objc_msgForward_impcache` 可以定位到`objc-msg-arm64.s` 汇编源码 如下：

```
STATIC_ENTRY __objc_msgForward_impcache

	// No stret specialization.
	b	__objc_msgForward

	END_ENTRY __objc_msgForward_impcache

	
	ENTRY __objc_msgForward

	adrp	x17, __objc_forward_handler@PAGE
	ldr	p17, [x17, __objc_forward_handler@PAGEOFF]
	TailCallFunctionPointer x17
	
	END_ENTRY __objc_msgForward
复制代码
```

-   关于 `forwardInvocation` 对应慢速消息转发 `methodSignatureForSelector` 方法签名：

> 1.  `forwardInvocation` 方法有两个任务: (1): 查找可以响应 `inInvocation` 中编码的消息的对象。对于所有消息，此对象不必相同。 (2): 使用 `anInvocation` 将消息发送到该对象。`anInvocation` 将保存结果，运行时系统将提取结果并将其传递给原始发送者。
> 2.  `forwardInvocation` 方法的实现不仅仅可以转发消息。`forwardInvocation`还可以用于合并响应各种不同消息的代码，从而避免了必须为每个选择器编写单独方法的麻烦。`forwardInvocation`方法在对给定消息的响应中还可能涉及其他几个对象，而不是仅将其转发给一个对象。
> 3.  `NSObject` 的 `forwardInvocation` 实现：只会调用 `dosNotRecognizeSelector`：方法，它不会转发任何消息。因此，如果选择不实现`forwardInvocation，将无法识别的消息发送给对象将引发异常。

下面举例来说明： 假设我们调用[dog walk]方法，那么它会经历如下过程：

> 1.  编译器会把`[dog walk]`转化为`objc_msgSend(dog，SEL)`，SEL为@selector(walk)。
> 2.  Runtime会在dog对象所对应的Dog类的方法缓存列表里查找方法的SEL
> 3.  如果没有找到，则在Dog类的方法分发表查找方法的SEL。（类由对象isa指针指向，方法分发表即methodList）
> 4.  如果没有找到，则在其父类（设Dog类的父类为Animal类）的方法分发表里查找方法的SEL（父类由类的superClass指向）
> 5.  如果没有找到，则沿继承体系继续下去，最终到达NSObject类。
> 6.  如果在234的其中一步中找到，则定位了方法实现的入口，执行具体实现
> 7.  如果最后还是没有找到，会面临两种情况：``(1) 如果是使用`［dog walk］`的方式调用方法````(2) 使用`［dog performSelector:@selector(walk)］`的方式调用方法`

如果是一个没有定义的方法，那么它会经历动态方法解析，消息转发流程

-   对象如何找到对应的方法去调用

> 1.  根据对象的isa去对应的类查找方法,isa:判断去哪个类查找对应的方法 指向方法调用的类
> 2.  根据传入的方法编号SEL，里面有个哈希列表，在列表中找到对应方法Method(方法名)
> 3.  根据方法名(函数入口)找到函数实现，函数实现在方法区

### 1.1.8 什么是method swizzling（俗称黑魔法）

简单说就是进行方法交换

> 1.  在Objective-C中调用一个方法，其实是向一个对象发送消息，查找消息的唯一依据是selector的名字。利用Objective-C的动态特性，可以实现在运行时偷换selector对应的方法实现，达到给方法挂钩的目的
> 2.  每个类都有一个方法列表，存放着方法的名字和方法实现的映射关系，selector的本质其实就是方法名，IMP有点类似函数指针，指向具体的Method实现，通过selector就可以找到对应的IMP
>     
>     ![[17169dce91f13a1d~tplv-t2oaga2asx-watermark.image.png]]
>     
> 3.  交换方法的几种实现方式： （1） 利用 method_exchangeImplementations 交换两个方法的实现 （2）利用 class_replaceMethod 替换方法的实现 （3）利用 method_setImplementation 来直接设置某个方法的IMP
>     
>     ![[17169dce97752030~tplv-t2oaga2asx-watermark.image.png]]
>     

-   方法交换实际应用：有一个需求:比如我有个项目,已经开发2年,之前都是使用UIImage去加载图片,组长想要在调用imageNamed,就给我提示,是否加载成功。

有三种方法解决这个需求问题：

> 1.  解决方式 自定义UIImage类，缺点：每次用要导入自己的类
> 2.  解决方法:UIImage分类扩充一个这样方法，缺点：需要导入，无法写super和self，会干掉系统方法，解决：给系统方法加个前缀，与系统方法区分，如：xmg_imageNamed：
> 3.  交互方法实现，步骤： 1.提供分类 2.写一个有这样功能方法 3.用系统方法与这个功能方法交互实现，在+load方法中实现

如果用方法2，每个调用imageNamed方法的，都要改成xmg_imageNamed：才能拥有这个功能，很麻烦。解决：用runtime交换方法 就比较好。

注意:在分类一定不要重写系统方法,就直接把系统方法干掉，如果真的想重写，在系统方法前面加前缀，方法里面去调用系统方法

思想:什么时候需要自定义,系统功能不完善,就自定义一个这样类,去扩展这个类

方法交换实现代码如下：

```
/#import "UIImage+Image.h"
/#import <objc/message.h>
@implementation UIImage (Image)
// 加载类的时候调用,肯定只会调用一次

 +(void)load
{
    // 交互方法实现xmg_imageNamed,imageNamed
    /**
     获取类方法名
     @param Class cls,#> 获取哪个类方法 description#>
     @param SEL name#> 方法编号 description#>
     @return 返回Method(方法名)
     class_getClassMethod(<#__unsafe_unretained Class cls#>, <#SEL name#>)
     */
    /**
     获取对象方法名
     @param Class cls,#> 获取哪个对象方法 description#>
     @param SEL name#> 方法编号 description#>
     @return 返回Method(方法名)
     class_getInstanceMethod(<#__unsafe_unretained Class cls#>, <#SEL name#>)
     */
    
   Method imageNameMethod = class_getClassMethod(self, @selector(imageNamed:));
    Method xmg_imageNameMethod = class_getClassMethod(self, @selector(xmg_imageNamed:));
    //用runtime对imageNameMethod和xmg_imageNameMethod方法进行交换
    method_exchangeImplementations(imageNameMethod, xmg_imageNameMethod);
}
//外界调用imageNamed：方法，其实是调用下面方法，调用xmg_imageNamed就是调用imageNamed：
+ (UIImage *)xmg_imageNamed:(NSString *)name
{
    //已经把xmg_imageNamed换成imageNamed，所以下面其实是调用的imageNamed：
   UIImage *image = [UIImage xmg_imageNamed:name];
    
    if (image == nil) {
        NSLog(@"加载失败");
    }
    return image;
}
@end
复制代码
```

## 1.2 结构模型

### 1.2.1 类结构，消息转发相关

上面主要是关于runtime流程的一些问题，接下来会有更加深入的需要深入理解objc4相关的源码。其中关于消息转发机制可能是最常见的问题了。

-   关于类结构，消息转发相关，你可能被问到的问题：

> 1.  介绍下runtime的内存模型（isa、对象、类、metaclass、结构体的存储信息等）
> 2.  为什么要设计metaclass
> 3.  class_copyIvarList & class_copyPropertyList区别
> 4.  class_rw_t 和 class_ro_t 的区别
> 5.  category如何被加载的,两个category的load方法的加载顺序，两个category的同名方法的加载顺序
> 6.  category & extension区别，能给NSObject添加Extension吗，结果如何
> 7.  消息转发机制，消息转发机制和其他语言的消息机制优劣对比
> 8.  在方法调用的时候，方法查询-> 动态解析-> 消息转发 之前做了什么
> 9.  IMP、SEL、Method的区别和使用场景
> 10.  load、initialize方法的区别什么？在继承关系中他们有什么区别
> 11.  说说消息转发机制的优劣

#### 1.2.1.1 类结构 isa指针相关问题

##### 1.2.1.1.1 说一下对 isa 指针的理解， 对象的isa 指针指向哪里？isa 指针有哪两种类型？

-   isa 等价于 is kind of

> 实例对象 isa 指向类对象 类对象指 isa 向元类对象 元类对象的 isa 指向元类的基类

-   isa 有两种类型

> 纯指针，指向内存地址 NON_POINTER_ISA，除了内存地址，还存有一些其他信息

-   isa源码分析 在Runtime源码查看isa_t是共用体。简化结构如下：

```
union isa_t 
{
    Class cls;
    uintptr_t bits;
    # if __arm64__ // arm64架构
#   define ISA_MASK        0x0000000ffffffff8ULL //用来取出33位内存地址使用（&）操作
#   define ISA_MAGIC_MASK  0x000003f000000001ULL
#   define ISA_MAGIC_VALUE 0x000001a000000001ULL
    struct {
        uintptr_t nonpointer        : 1; //0：代表普通指针，1：表示优化过的，可以存储更多信息。
        uintptr_t has_assoc         : 1; //是否设置过关联对象。如果没设置过，释放会更快
        uintptr_t has_cxx_dtor      : 1; //是否有C++的析构函数
        uintptr_t shiftcls          : 33; // MACH_VM_MAX_ADDRESS 0x1000000000 内存地址值
        uintptr_t magic             : 6; //用于在调试时分辨对象是否未完成初始化
        uintptr_t weakly_referenced : 1; //是否有被弱引用指向过
        uintptr_t deallocating      : 1; //是否正在释放
        uintptr_t has_sidetable_rc  : 1; //引用计数器是否过大无法存储在ISA中。如果为1，那么引用计数会存储在一个叫做SideTable的类的属性中
        uintptr_t extra_rc          : 19; //里面存储的值是引用计数器减1

#       define RC_ONE   (1ULL<<45)
#       define RC_HALF  (1ULL<<18)
    };

# elif __x86_64__ // arm86架构,模拟器是arm86
#   define ISA_MASK        0x00007ffffffffff8ULL
#   define ISA_MAGIC_MASK  0x001f800000000001ULL
#   define ISA_MAGIC_VALUE 0x001d800000000001ULL
    struct {
        uintptr_t nonpointer        : 1;
        uintptr_t has_assoc         : 1;
        uintptr_t has_cxx_dtor      : 1;
        uintptr_t shiftcls          : 44; // MACH_VM_MAX_ADDRESS 0x7fffffe00000
        uintptr_t magic             : 6;
        uintptr_t weakly_referenced : 1;
        uintptr_t deallocating      : 1;
        uintptr_t has_sidetable_rc  : 1;
        uintptr_t extra_rc          : 8;
#       define RC_ONE   (1ULL<<56)
#       define RC_HALF  (1ULL<<7)
    };

# else
#   error unknown architecture for packed isa
# endif

}
复制代码
```

-   继续查看结构体objc_class的定义:

```
typedef struct objc_class *Class;

struct objc_class {

    Class isa  OBJC_ISA_AVAILABILITY;   //isa指针，指向metaclass（该类的元类）

#if !__OBJC2__

    Class super_class   //指向objc_class（该类）的super_class（父类）

    const char *name    //objc_class（该类）的类名

    long version        //objc_class（该类）的版本信息，初始化为0，可以通过runtime函数class_setVersion和class_getVersion进行修改和读取

    long info           //一些标识信息，如CLS_CLASS表示objc_class（该类）为普通类。ClS_CLASS表示objc_class（该类）为metaclass（元类）

    long instance_size  //objc_class（该类）的实例变量的大小

    struct objc_ivar_list *ivars    //用于存储每个成员变量的地址

    struct objc_method_list **methodLists   //方法列表，与info标识关联

    struct objc_cache *cache        //指向最近使用的方法的指针，用于提升效率

    struct objc_protocol_list *protocols    //存储objc_class（该类）的一些协议

#endif

} OBJC2_UNAVAILABLE;

typedef struct objc_object *id;

struct objc_object {

    Class isa  OBJC_ISA_AVAILABILITY;

};


复制代码
```

> struct `objc_classs`结构体里存放的数据称为元数据(`metadata`)，通过成员变量的名称我们可以猜测里面存放有指向父类的指针、类的名字、版本、实例大小、实例变量列表、方法列表、缓存、遵守的协议列表等，这些信息就足够创建一个实例了，该结构体的第一个成员变量也是`isa`指针，这就说明了`Class`本身其实也是一个对象，我们称之为类对象，类对象在编译期产生用于创建实例对象，是单例。

`objec_object`（对象）中isa指针指向的类结构称为`objec_class`（该对象的类），其中存放着普通成员变量与对象方法 （“-”开头的方法）。

`objec_class`（类）中`isa`指针指向的类结构称为`metaclass`（该类的元类），其中存放着`static`类型的成员变量与`static`类型的方法 （“+”开头的方法）。

元类(`metaclass`)：在oc中，每一个类实际上也是一个对象。也有一个isa指针。因为类也是一个对象，所以也必须是另外一个类的实例，这个类就是元类(`metaclass`)，即元类的对象就是这个类，元类保存了类方法的列表

在oc语言中，每一个类实际上也是一个对象。每一个类也有一个isa指针。每一个类也可以接收消息。因为类也是一个对象，所以也是另外一个类的实例，这个类就是元类(`metaclass`)。元类也是一个对象，所有的元类的isa指针都会指向一个根元类。根元类的isa指针又会指向他自己，这样就形成了一个闭环。

如下图展示了类的继承和isa指向的关系：

![[1716c8f4c09578f2~tplv-t2oaga2asx-watermark.image.png]]

**isa指针的指向**：

> 1.  一个objc对象的`isa`指针指向他的类对象，类对象的isa指针指向他的元类，元类的isa指针指向根元类，所有的元类isa都指向同一个根元类，根元类的isa指针指向根元类本身。根元类`super class`父类指向`NSObject`类对象。根`metaclass`（元类）中的`superClass`指针指向根类，因为根`metaclass`（元类）是通过继承根类产生的。
> 2.  实例对象的`isa`指针， 指向他的类对象，类对象的`isa` 指针， 指向他的元类。系统判断一个对象属于哪个类，也是通过这个对象的`isa`指针的指向来判断。对象中的成员变量，存储在对象本身，对象的实例方法，存储在他的`isa` 指针所指向的对象中。
> 3.  对象在调用减号方法的时候，系统会在对象的`isa`指针所指向的类对象中找方法，这一段在`kvo`的实现原理中就能看到，`kvo`的实现原理就是系统动态的生成一个类对象，这个类是监听对象的类的子类，在生成的子类中重写了监听属性的set方法，之后将监听对象的`isa`指针指向系统动态生成的这个类，当监听对象调用set方法时，由于监听对象的`isa`指针指向的是刚刚动态生成的类，所以在执行的set方法也是子类中重写的set方法，这就是kvo的实现原理。同理，我们也可以通过`runtime`中的方法设置某个对象`isa`指针指向的类对象，让对象调用一些原本不属于他的方法。

-   **类对象（class object）**:

> 1.  类对象由编译器创建。任何直接或间接继承了`NSObject`的类，它的实例对象中都有一个`isa`指针，指向它的类对象。这个类对象中存储了关于这个实例对象所属的类的定义的一切：包括变量，方法，遵守的协议等等。因此，类对象能访问所有关于这个类的信息，利用这些信息可以产生一个新的实例，但是类对象不能访问任何实例对象的内容。类对象没有自己的实例变量。

例如：我们创建p对象：`Person * p = [Person new] ;`

在创建一个对象p之前，在堆内存中就先存在了一个类(Person)（类对象），类对象在编绎时系统会为我们自动创建。在类第一次加载进内存时创建。

创建一个对象之后，在堆内存中会创建了一个p对象，该对象包含了一个`isa`指针的成员变量（第一个属性），`isa`指针则指向在堆里面存在的类对象， 在栈内存里创建了一个该类的指针p，p指针指向的是`isa`地址,`isa`指向Person.

-   **对象方法的调用：**

> OC调用方法，运行的时候编译器会将代码转化为`objc_msgSend(obj, @selector (selector))`，在`objc_msgSend`函数中首先通过`obj`（对象）的`isa`指针找到`obj`（对象）对应的class（类）。在`class`（类）中先去`cache`中通过SEL（方法的编号）查找对应method（方法），若cache中未找到，再去`methodLists`中查找，若`methodists`中未找到，则去`superClass`中查找，若能找到，则将method（方法）加入到cache中，以方便下次查找，并通过method（方法）中的函数指针跳转到对应的函数中去执行。如果仍然找不到，则继续通过 super_class向上一级父类结构体中查找，直至根class

-   **类方法的调用：**

> 1.  当我们调用某个类方法时，它首先通过自己的`isa`指针指向的`objc_class`中的`isa`指针找到元类，并从其`methodLists`中查找该类方法，如果找不到则会通过元类）的super_class指针找到父类的元类结构体，然后从`methodLists`中查找该方法，如果仍然找不到，则继续通过`super_class`向上一级父类结构体中查 找，直至根元类；
> 2.  C语言函数编译的时候就会决定调用哪个函数，OC是一种动态语言，他会尽可能把代码的从编译链接推迟到运行时，这就是oc运行时多态。 给一个对象发送消息，并不会立即执行，而是在运行的时候在去寻找他对应的实现而OC的函数，属于动态调用过程，在编译期并不能决定真正调用哪个函数，只有在真正运行时才会根据函数的名称找到对应的函数来调用。

## 1.3 内存管理相关

参考：[www.jianshu.com/p/8345a79fd…](https://link.juejin.cn/?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2F8345a79fd572 "https://www.jianshu.com/p/8345a79fd572") [juejin.cn/post/684490…](https://juejin.cn/post/6844903971811753998 "https://juejin.cn/post/6844903971811753998") [github.com/ChenYilong/…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2FChenYilong%2FiOSInterviewQuestions%2Fblob%2Fmaster%2F01%25E3%2580%258A%25E6%258B%259B%25E8%2581%2598%25E4%25B8%2580%25E4%25B8%25AA%25E9%259D%25A0%25E8%25B0%25B1%25E7%259A%2584iOS%25E3%2580%258B%25E9%259D%25A2%25E8%25AF%2595%25E9%25A2%2598%25E5%258F%2582%25E8%2580%2583%25E7%25AD%2594%25E6%25A1%2588%2F%25E3%2580%258A%25E6%258B%259B%25E8%2581%2598%25E4%25B8%2580%25E4%25B8%25AA%25E9%259D%25A0%25E8%25B0%25B1%25E7%259A%2584iOS%25E3%2580%258B%25E9%259D%25A2%25E8%25AF%2595%25E9%25A2%2598%25E5%258F%2582%25E8%2580%2583%25E7%25AD%2594%25E6%25A1%2588%25EF%25BC%2588%25E4%25B8%258B%25EF%25BC%2589.md "https://github.com/ChenYilong/iOSInterviewQuestions/blob/master/01%E3%80%8A%E6%8B%9B%E8%81%98%E4%B8%80%E4%B8%AA%E9%9D%A0%E8%B0%B1%E7%9A%84iOS%E3%80%8B%E9%9D%A2%E8%AF%95%E9%A2%98%E5%8F%82%E8%80%83%E7%AD%94%E6%A1%88/%E3%80%8A%E6%8B%9B%E8%81%98%E4%B8%80%E4%B8%AA%E9%9D%A0%E8%B0%B1%E7%9A%84iOS%E3%80%8B%E9%9D%A2%E8%AF%95%E9%A2%98%E5%8F%82%E8%80%83%E7%AD%94%E6%A1%88%EF%BC%88%E4%B8%8B%EF%BC%89.md")[www.jianshu.com/p/0bf8787db…](https://link.juejin.cn/?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2F0bf8787db4c3 "https://www.jianshu.com/p/0bf8787db4c3")

  
作者：孔雨露  
链接：https://juejin.cn/post/6844904121531809806  
来源：掘金  
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。