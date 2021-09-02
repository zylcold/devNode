dyld2 和 dyld3 的主要区别就是没有启动闭包，就导致每次启动都要：

* 解析动态库的依赖关系
* 解析 LINKEDIT，找到 bind & rebase 的指针地址，找到 bind 符号的地址
* 注册 objc 的 Class/Method 等元数据，对大型工程来说，这部分耗时会很长