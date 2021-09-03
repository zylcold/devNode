关于 isa 名字的由来, 英文是 “is a xxx”



##  isa 详解
* 在 arm64 之前 isa 只一个指针,存着 class 或 meta-class 的内存地址
* 在 arm64 之后, isa 是一个 union, 使用位域来储存更多东西
![[Pasted image 25.png]]
![[Pasted image 26.png]]

## isa、 superClass 总结

需要注意两点:
1. metaclass 的 isa 都指向 nsobject 的 meta class
2. nsobject 的 meta class 的 superClass 指向 nsobject 的类对象

	![[img_isa和superClass关系图.png]]


## isa 指针和方法的调用
![[Pasted image.png]]
