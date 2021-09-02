# 什么是符号表
符号表就是指在Xcode项目编译后，在编译生成的二进制文件.app的同级目录下生成的同名的.dSYM文件，一般位于 /Users/<用户名>/Library/Developer/Xcode/Archives 目录下。

`.dSYM`文件其实是一个目录，在子目录中包含了一个16进制的保存函数地址映射信息的中转文件，所有Debug的symbols都在这个文件中(包括文件名、函数名、行号等)，所以也称之为调试符号信息文件。


# 如何获取符号表dSYM
一般位于 /Users/<用户名>/Library/Developer/Xcode/Archives 

对于每一个发布版本我们都很有必要保存对应的 Archives 文件 ( [AUTOMATICALLY SAVE THE DSYM FILES](http://www.cimgf.com/2009/12/23/automatically-save-the-dsym-files/) 这篇文章介绍了通过脚本每次编译后都自动保存 dSYM 文件)。


XCode中，选择Window -> Organizer 可以看到我们生成的archier文件

![[20161203182006555.jpeg]]

* 右键 -> 在finder中显示。
* 右键 -> 查看包内容。
* 找到dSYM


# 符号表有什么用

符号表就是用来符号化 crash log（崩溃日志）。crash log中有一些方法16进制的内存地址等，通过符号表就能找到对应的能够直观看到的方法名之类。


# 如何将文件一一对应

每一个 xx.app 和 xx.app.dSYM 文件都有对应的 UUID，crash 文件也有自己的 UUID，只要这三个文件的 UUID 一致，我们就可以通过他们解析出正确的错误函数信息了。

1. 查看 xx.app 文件的 UUID，terminal 中输入命令 ：
```shell
dwarfdump --uuid xx.app/xx (xx代表你的项目名)
```

2. 查看 xx.app.dSYM 文件的 UUID ，在 terminal 中输入命令：
```shell
dwarfdump --uuid xx.app.dSYM 
```

3.crash 文件内第一行 Incident Identifier 就是该 crash 文件的 UUID。


# 如何使用.dsYM

1. 友盟.dsYM分析

如果是使用友盟的话，我们能在错误列表里看到一些错误，然后可以导出奔溃信息，导出的文件为.csv文件。友盟有一个分析工具，使用那个工具可以看到一些错误的函数，行号等。但是很容易分析失败，不知道为什么？

注意：使用的时候要确保你的.xcarchive在 ~/Library/Developer/Xcode/或该路径的子目录下。

.xcarchive里的.dsYM文件和.app文件是有对应的UUID的。然后你的错误详情里也是有UUID，只有当UUID相等时才能分析对。

我犯的错误：因为我们是两个人开发，Archive的时候都是在另一个人的电脑上Archive的，所以我的电脑里根本没有对应的.xcarchive文件。所以我在我电脑上用友盟的分析工具分析是时候是监测不出来错误的。

2. 第三方小工具.dsYM分析

或者自己找到.xcarchive文件和错误内存地址(友盟错误详情里标绿色的为错误内存地址)。然后通过一个小应用来分析出对应的函数。 [应用下载地址](http://pan.baidu.com/s/1bnkxPvT),具体可参考文章 [dSYM 文件分析工具](http://www.cocoachina.com/ios/20141219/10694.html)

意拿来分析的xcarchive名字不要有空格或特殊字符，直接用最简单的数字就好了。

3. [[Crash日志符号化]]