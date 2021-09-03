## 基础知识
1. 会在 class 第一次接收消息时调用
2. 调用时,如果父类没有初始化,会先调用父类的初始化方法,见`objc-initialize.m` 中 `_class_initialize`方法
3. 调用顺序,先父类,后子类

![[Pasted image 10.png]]

## 相关问题
#### initialize 和 [[load方法]] 的区别
1. 调用方式
	* load是根据函数地址直接调用
	* initialize是通过objc_msgSend调用
2. 调用时机
	* load是runtime加载类、分类的时候调用（只会调用1次）
	* initialize是类第一次接收到消息的时候调用，每一个类只会initialize一次
		* 如果子类没实现initialize, 就会调用父类的,所以父类 initialize 可能会被调用多次
		* category 实现了 initialize, 就会覆盖 class 本身的 initialize

#### load、initialize的调用顺序？
* load
	![[load方法调用过程]]
* initialize
		* 先初始化父类
		* 在初始化子类(可能最终调用的是父类的 initialize方法)