## iOS 用什么方式实现[[KVO]]?([[KVO的本质]]是什么?)

利用Run time 动态生成一个 被监听者的子类
当修改对象属性时,会调用 Foundation  的  NSSetXXXValueAndNotify,该方法会调用
	willchangevalueForkey:
	父类原来的 setter
	didchangeValueForkey:
	会触发监听者的	observeValueForKeyPath:ofObject:change:context: 方法
	
## 如何手动调用KVO
	
⼿动调⽤willChangeValueForKey:和didChangeValueForKey:


## 直接修改成员变量会触发 KVO 吗

不会,因为没有调用 setter 方法



