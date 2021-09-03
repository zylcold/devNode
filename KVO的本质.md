利用Run time 动态生成一个 被监听者的子类
当修改对象属性时,会调用 Foundation  的  NSSetXXXValueAndNotify,该方法会调用
	willchangevalueForkey:
	父类原来的 setter
	didchangeValueForkey:
	会触发监听者的	observeValueForKeyPath:ofObject:change:context: 方法
	
	
可以对比如下两张图看本质
![[Pasted image 2.png]]

![[Pasted image 3.png]]