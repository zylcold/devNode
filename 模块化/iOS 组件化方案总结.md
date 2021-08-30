#module

## 开源组件化框架推荐（MJRouter、CTMediator、BeeHive）

目前主流的组件化方式有三种

* `URL` 路由

* `target-action`

* `protocol` 匹配

下面介绍一下三种方式对应的代表性框架

## MJRouter（URL路由）

iOS常用路由工具一般都是使用URL匹配，或者根据约定的命名使用runtime方法进行动态调用。

这种实现方式优点是实现简单，缺点是需要维护字符串表，或者根据依赖命名约定无法在编译时暴露问题，运行时才发现错误。

URL 路由方式典型代表框架是蘑菇街开源的  [MGJRouter](https://github.com/meili/MGJRouter) ，还有  [JLRoutes](https://github.com/joeldev/JLRoutes) ， [HHRouter](https://github.com/Huohua/HHRouter)  等

### MGJRouter 实现分析：

1. App 启动时实例化各组件模块或者使用class注册，然后组件向 `ModuleManager` 注册 `Url`
2. 当组件A需要调用组件B时，向 `ModuleManager` 传递URL，参数可以拼接在URL后面或者放在字典里传递，类似 `openURL` 。然后由 `ModuleManager` 负责调度组件B，最后完成目标

注册及调用代码如下：

```objc
// 1、注册某个URL
[MGJRouter registerURLPattern:@"mgj://foo/bar" toHandler:^(NSDictionary *routerParameters) {
    NSLog(@"routerParameterUserInfo:%@", routerParameters[MGJRouterParameterUserInfo]);
}];
//2、调用路由[MGJRouter openURL:@"mgj://foo/bar"];

[MGJRouter registerURLPattern:@"mgj://category/travel" toHandler:^(NSDictionary *routerParameters) {
    NSLog(@"routerParameters[MGJRouterParameterUserInfo]:%@", routerParameters[MGJRouterParameterUserInfo]);
    // @{@"user_id": @1900}
}];

[MGJRouter openURL:@"mgj://category/travel" withUserInfo:@{@"user_id": @1900} completion:nil];

```

**URL 路由的优点**

* 具有很高的动态性，适合经常开展运营活动的app，例如蘑菇街自身属于电商行业

* 方便统一管理多平台的路由规则

* 易于适配 URL Scheme

**URl 路由的缺点**

* 传参方式有限，并且无法利用编译器进行参数类型检查，因此所有的参数都是通过字符串转换而来

* 只适用于界面模块，不适用于通用模块

* 参数的格式不明确，是个灵活的 dictionary，也需要有个地方可以查参数格式。

* 不支持storyboard

* 依赖于字符串硬编码，难以管理，蘑菇街做了个后台专门管理。

* 无法保证所使用的的模块一定存在

* 解耦能力有限，url 的”注册”、”实现”、”使用”必须用相同的字符规则，一旦任何一方做出修改都会导致其他方的代码失效，并且重构难度大

## CTMediator（target-action）

target-action 方案是基于OC的runtime、category特性动态获取模块，例如通过 `NSClassFromString` 获取类并创建实例，通过 `performSelector + NSInvocation` 动态调用方法，典型代表框架是 [CTMediator](https://github.com/casatwy/CTMediator)

### 实现分析：

1. 利用分类为路由添加新接口，在接口中通过字符串获取对应的类

2. 通过runtime创建实例，动态调用实例的方法

页面间跳转实现代码如下：

```objc
//******* 1、分类定义新接口
#import "CTMediator+A.h"

@implementation CTMediator (A)

- (UIViewController *)A_Category_Swift_ViewControllerWithCallback:(void (^)(NSString *))callback
{
    NSMutableDictionary *params = [[NSMutableDictionary alloc] init];
    params[@"callback"] = callback;
    params[kCTMediatorParamsKeySwiftTargetModuleName] = @"A_swift";
    return [self performTarget:@"A" action:@"Category_ViewController" params:params shouldCacheTarget:NO];
}

- (UIViewController *)A_Category_Objc_ViewControllerWithCallback:(void (^)(NSString *))callback
{
    NSMutableDictionary *params = [[NSMutableDictionary alloc] init];
    params[@"callback"] = callback;
    return [self performTarget:@"A" action:@"Category_ViewController" params:params shouldCacheTarget:NO];
}
@end

//******* 2、模块提供者提供target-action的调用方式（对外需要加上public关键字）
#import "Target_A.h"
#import "AViewController.h"

@implementation Target_A

- (UIViewController *)Action_Category_ViewController:(NSDictionary *)params
{
    typedef void (^CallbackType)(NSString *);
    CallbackType callback = params[@"callback"];
    if (callback) {
        callback(@"success");
    }
    AViewController *viewController = [[AViewController alloc] init];
    return viewController;
}
- (UIViewController *)Action_Extension_ViewController:(NSDictionary *)params
{
    typedef void (^CallbackType)(NSString *);
    CallbackType callback = params[@"callback"];
    if (callback) {
        callback(@"success");
    }
    AViewController *viewController = [[AViewController alloc] init];
    return viewController;
}

//******* 3、使用
// Objective-C -> Category -> Objective-C
UIViewController *viewController = [[CTMediator sharedInstance] A_Category_Objc_ViewControllerWithCallback:^(NSString *result) {
    NSLog(@"%@", result);
}];
[self.navigationController pushViewController:viewController animated:YES];

```

**优点**

* 利用 `分类` 可以明确声明接口，进行编译检查

* 实现方式 `轻量`

**缺点**

* 需要在 `mediator` 和 `target` 中重新添加每一个接口，模块化时代码较为繁琐

* 在 `category` 中仍然引入了 `字符串硬编码` ，内部使用字典传参

* 无法保证使用的模块一定存在，target在修改后，使用者只能在运行时才能发现错误

* 可能会创建过多的 target 类

**CTMediator 核心源码分析**

> 通过分类中调用的 `performTarget` 来到 `CTMediator` 中的具体实现，即 `performTarget:action:params:shouldCacheTarget:` ，主要是通过传入的name，找到对应的 `target` 和 `action`

```objc
- (id)performTarget:(NSString *)targetName action:(NSString *)actionName params:(NSDictionary *)params shouldCacheTarget:(BOOL)shouldCacheTarget
{
    if (targetName == nil || actionName == nil) {
        return nil;
    }
    //在swift中使用时，需要传入对应项目的target名称，否则会找不到视图控制器
    NSString *swiftModuleName = params[kCTMediatorParamsKeySwiftTargetModuleName];
    
    // generate target 生成target
    NSString *targetClassString = nil;
    if (swiftModuleName.length > 0) {
        //swift中target文件名拼接
        targetClassString = [NSString stringWithFormat:@"%@.Target_%@", swiftModuleName, targetName];
    } else {
        //OC中target文件名拼接
        targetClassString = [NSString stringWithFormat:@"Target_%@", targetName];
    }
    //缓存中查找target
    NSObject *target = [self safeFetchCachedTarget:targetClassString];
    //缓存中没有target
    if (target == nil) {
        //通过字符串获取对应的类
        Class targetClass = NSClassFromString(targetClassString);
        //创建实例
        target = [[targetClass alloc] init];
    }

    // generate action 生成action方法名称
    NSString *actionString = [NSString stringWithFormat:@"Action_%@:", actionName];
    //通过方法名字符串获取对应的sel
    SEL action = NSSelectorFromString(actionString);
    
    if (target == nil) {
        // 这里是处理无响应请求的地方之一，这个demo做得比较简单，如果没有可以响应的target，就直接return了。实际开发过程中是可以事先给一个固定的target专门用于在这个时候顶上，然后处理这种请求的
        [self NoTargetActionResponseWithTargetString:targetClassString selectorString:actionString originParams:params];
        return nil;
    }
    //是否需要缓存
    if (shouldCacheTarget) {
        [self safeSetCachedTarget:target key:targetClassString];
    }
    //是否响应sel
    if ([target respondsToSelector:action]) {
        //动态调用方法
        return [self safePerformAction:action target:target params:params];
    } else {
        // 这里是处理无响应请求的地方，如果无响应，则尝试调用对应target的notFound方法统一处理
        SEL action = NSSelectorFromString(@"notFound:");
        if ([target respondsToSelector:action]) {
            return [self safePerformAction:action target:target params:params];
        } else {
            // 这里也是处理无响应请求的地方，在notFound都没有的时候，这个demo是直接return了。实际开发过程中，可以用前面提到的固定的target顶上的。
            [self NoTargetActionResponseWithTargetString:targetClassString selectorString:actionString originParams:params];
            @synchronized (self) {
                [self.cachedTarget removeObjectForKey:targetClassString];
            }
            return nil;
        }
    }
}

```

> 进入 `safePerformAction:target:params:` 实现，主要是通过 `invocation` 进行 `参数传递+消息转发`

```objc
- (id)safePerformAction:(SEL)action target:(NSObject *)target params:(NSDictionary *)params
{
    //获取方法签名
    NSMethodSignature* methodSig = [target methodSignatureForSelector:action];
    if(methodSig == nil) {
        return nil;
    }
    //获取方法签名中的返回类型，然后根据返回值完成参数传递
    const char* retType = [methodSig methodReturnType];
    //void类型
    if (strcmp(retType, @encode(void)) == 0) {
        ...
    }
    //...其他类型可以阅读CTMediator源码
}

```

## BeeHive（protocol class）

protocol匹配的 `实现思路` 是：

* 1、将 `protocol` 和对应的 `类` 进行 `字典匹配`

* 2、通过用 `protocol` 获取 `class` ，在 `动态创建实例`

protocol比较典型的三方框架就是 [阿里的BeeHive](https://github.com/alibaba/BeeHive) 。 `BeeHive` 借鉴了Spring Service、Apache DSO的架构理念， `采用AOP+扩展App生命周期API` 形式，将 `业务功能`、`基础功能` 模块以模块方式以解决大型应用中的复杂问题，并让 `模块之间以Service形式调用` ，将复杂问题切分，以AOP方式模块化服务。

BeeHive 源码还没有深入探究，想了解的可以参考链接： [BeeHive -- 一次iOS模块化解耦实践](https://halfrost.com/beehive/)

附参考文章：

[打造完备的iOS组件化方案：如何面向接口进行模块解耦](https://zuikyo.github.io/2019/07/15/iOS_inrerface_orientation_modularization/)

[iOS 组件化方案总结](https://juejin.cn/post/6921970988796248077)