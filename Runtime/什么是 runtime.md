## 什么是 runtime?
runtime 支撑和实现了 objective-c 的动态性,允许很多操作推迟到程序运行时在进行
runtime 是一套 C 语言的API,封装了很多动态性相关的函数
平时编写的 objective-c 代码,底层都是转换成了 runtime 进行调用

![[Pasted image 24.png]]

## objc_msgSend 执行流程
[[objc_msgSend]] 
