#exception

非法参数异常(NSInvalidArgumentException) 是 Objective - C 代码最常出现的错误，所以平时在写代码的时候，需要多加注意，加强对参数的检查，避免传入非法参数导致异常，其中尤以nil参数为甚。

1. 集合数据的参数传递
	比如NSMutableArray, NSMutableDictionary的数据操作
		(1) NSDictionary不能删除nil的key
		(2) NSDictionary不能添加nil的对象
		(3) 不能插入nil的对象
		(4) 其他一些nil参数

2. 其他一些API的使用
	APP一般都会有网络操作，免不了使用网络相关接口，比如NSURL的初始化，不能传入nil的http地址:

3. 未实现的方法
	(1) .h文件里函数名，却忘了修改.m文件里对应的函数名
	(2) 使用第三方库时，没有添加”-ObjC” flag
	(3) MRC时，大部分情况下是因为对象被提前release了，在你心里不希望他release的情况下，指针还在，对象已经不在 了。