

## 一个 NSObject 对象占用多少内存
- 通过 malloc_size() 得知系统分配了16个字节给该对象
- 但是通过 class_getInstanceSize() 得知该对象只占用了 8 个字节
- malloc_size() 返回16 字节是为了[[内存对齐]]


## 对象的 [[isa指针]]指向哪里

instance 的指向 class
class 的 指向 metaClass
metaClass 的 指向 NSObject 的 metaClass

![[img_isa和superClass关系图.png]]


## oc 的类信息存放在哪里

instance 的变量(属性)的具体值放在 instance 对象中
实例方法、属性信息、成员变量信息、协议 放在[[类对象]]
类方法,存在 [[metaClass]] 中







