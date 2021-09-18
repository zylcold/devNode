#runtime  

## super 的本质
super 的本质其实就是在[[消息发送]]时,跳过从 instance 本身的类对象中去查找方法列表,而是直接从类对象的父类开始查找方法列表.

所以消息的接受者仍然是 instance 对象
![[_Users_zhuyunlong_Documents_软件开发_attachments_Pasted image 23.png]]

![[Pastedimage22.png]]

## 相关面试题
#### 题目一
person是父类, MJStudent 继承 person,
调用 MJStudent 的 print 方法会打印什么?

答案是 MJStudent 自己的 name
```objectivec
@interface MJPerson : NSObject
@property (copy, nonatomic) NSString *name;
- (void)print;
@end

@implementation MJPerson

- (void)print
{
    NSLog(@"my name is %@", self->_name);
}

@end
```

```objectivec

@interface MJStudent : MJPerson
@end

@implementation MJStudent

- (void)print
{
    [super print];
}

@end
```

#### 题目二

前提条件一样,下面会打印什么
```objectivec

@interface MJStudent : MJPerson
@end

@implementation MJStudent

- (instancetype)init {
    if (self = [super init]) {
        NSLog(@"[self class] = %@", [self class]);              // MJStudent
        NSLog(@"[super class] = %@", [super class]);            // MJStudent
        NSLog(@"[self superclass] = %@", [self superclass]);    // MJPerson
        NSLog(@"[super  superclass] = %@", [super  superclass]);// MJPerson
    }
    return self;
}
@end

```

打印结果
```shell
[self class] = MJStudent
[super class] = MJStudent
[self superclass] = MJPerson
[super  superclass] = MJPerson
```

#### 题目三

下面代码会打印什么,注意调用的主体是类对象

```objectivec
NSLog(@"%d", [NSObject isKindOfClass:[NSObject class]]); 
NSLog(@"%d", [NSObject isMemberOfClass:[NSObject class]]); 
NSLog(@"%d", [MJPerson isKindOfClass:[MJPerson class]]); 
NSLog(@"%d", [MJPerson isMemberOfClass:[MJPerson class]]);
```

答案是只有 `[NSObject isKindOfClass:[NSObject class]]` 为真,其他为否.
原因是 NSObject 会在元类中找方法, NSObject的元类没有isKindOfClass, 再去元类的父类中,也就是 NSObject 类对象找到方法,具体可参考[[isa指针]]