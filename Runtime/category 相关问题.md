## [[category]] 使用的场合是什么

给系统或三方的类添加方法
给原有类添加方法, 拆分代码,使逻辑清楚


## [[category]] 的实现原理

category 编译之后(c++重新编译) 的底层结构是 struct [[category_t]]  里面存储着分类的对象方法、类方法、协议、属性信息

程序运行的时候, runtime  会将Category 的的数据,合并到类信息中(类对象、元类中)

![[runtime加载category的过程]]


## [[category]] 和类的 extension 区别是什么

class 的 extension 在编译的时候,数据就已经包含在类信息中

category 在运行时,才会将数据合并到类信息中

![[runtime加载category的过程]]


## [[category]] 中有 [[load方法]] 吗?
有 


## [[load方法]] 是什么时候调用的 
在 runtime 加载类、分类的时候调用

![[load方法调用过程]]


## [[load方法]] 能继承吗
load 方法可以继承,但一搬情况不会主动调用,都是让系统自动调用


## [[load方法]] [[initialize方法]] 的区别是什么? 他们在catagory中的调用顺序?出现继承时他们的调用过程

* 区别: 
	* 加载时机: load 在程序启动调用 [[load方法调用过程]],initialize 在类第一次收到消息时调用
	* 调用次数: load 只会调用一次,initialize 可能会调用多次
* 调用顺序:
	* load: 父类、子类、分类
	* initialize:父类、子类(分类实现了就会调用分类而不是子类)
* 出现继承时调用过程:
	* load
		* 先调用父类(递归查找,看源码可知)
		* 在调用子类
		* 在调用分类
	* initialize
		* 如果子类没有实现,会调用父类的,父类initialize 可能调用多次
		* 如果 cagetory 实现了,那么就不会调用 class 的 initialize


## [[category]]  能否添加成员变量? 如何给category 添加成员变量

不可以添加,可以通过[[关联对象]]间接实现添加后的效果








	


