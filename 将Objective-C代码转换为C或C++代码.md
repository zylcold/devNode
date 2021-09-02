
指令如下:
```shell
xcrun  -sdk  iphoneos  clang  -arch  arm64  -rewrite-objc  OC源⽂件  -o  输出的CPP⽂件

```
如果需要链接其他框架，使⽤-framework参数。⽐如-framework UIKit