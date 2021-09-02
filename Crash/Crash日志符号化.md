#crash 

# 堆栈符号解析

分为两种情景讨论：

1、Crash收集上报的堆栈信息解析
这种信息一般还原度比较高，基本上都能给出崩溃的具体定位信息，一些需要解析堆栈符号的解决方法

异常信息有三种类型：
1. 已标记错误位置的:
```
test 0x000000010bfddd8c -[ViewController viewDidLoad] + 8588
```
这种信息已经很明确了，不用解析

2. 有模块地址的情况：
```
test 0x00000001018157dc 0x100064000 + 24844252
```

以上面为例子，从左到右依次是：

二进制库名(test) 调用方法的地址(0x00000001018157dc) 模块地址(0x100064000) +偏移地址(24844252)

3. 无模块地址的情况：
```
test 0x00000001018157dc test + 24844252
```

无模块地址的情况下计算模块地址

1. 先将偏移地址转为16进制：

		24844252 = 0x17B17DC

2. 然后用方法的地址-偏移地址，得到的就是模块地址

		0x00000001018157dc - 0x17B17DC = 0x100064000


[[符号表dSYM#如何获取符号表dSYM|获取符号表dSYM]]

# 解析工具
## symbolicatecrash
1. 将symbolicatecrash、.dSYM、.app、crash.crash拷贝到桌面下同一个文件夹下

2. 检查 xx.app 和 xx.app.dSYM 文件以及crash 文件这三种的 UUID是否一致。
	
	* 查看 xx.app 文件的 UUID，terminal 中输入命令 ：
	* 
```shell
dwarfdump --uuid xx.app/xx (xx代表你的项目名)
```
* 查看 xx.app.dSYM 文件的 UUID ，在 terminal 中输入命令：

```shell
dwarfdump --uuid xx.app.dSYM
```

* 查看crash 日志中的Incident Identifier （crash 文件的 UUID）

3. 使用命令，生成“可定位问题的crash文件”

```shell
//symbolreportXXX.crash就是符号化后的文件
./symbolicatecrash crashXXX.crash appName.app.dSYM > symbolreportXXX.crash
```

4. 根据符号化后的线程回溯信息，可以帮助定位出问题的代码行。

**说明** ：如果执行symbolicatecrash命令出现 *Error: "DEVELOPER_DIR" is not defined at ./symbolicatecrash...* 这样的错误，可以在执行命令前，输入 **export DEVELOPER_DIR="/Applications/XCode.app/Contents/Developer"**

## atos命令

```shell
atos [-arch 架构名] [-o 符号表] [-l 模块地址] [方法地址]
```

在符号化时候，还可以使用atos命令。发现armv7处理器上的crash使用symbolicatecrash无法符号化。

1. 将.dSYM、.app、crash.crash放到同一个文件夹下。

2. 知道crash文件的UUID：执行grep "AppName arm" *crash，得到结果

```shell
crash1.crash:0x100040000 - 0x100e23fff +AppName arm64 <ba0e190dcd1b37349e1362be7e9b7e62> /var/containers/Bundle/Application/55A4D641-847F-4D24-86E1-129B28461858/AppName.app/AppName
crash2.crash:0x100060000 - 0x100e43fff +AppName arm64 <ba0e190dcd1b37349e1362be7e9b7e62> /var/containers/Bundle/Application/3229ED68-8D19-406D-A3F5-EC0310C9DB7C/QAppName.app/AppName
crash3.crash:    0x5000 -   0xce8fff +AppName armv7 <7d62327effef37d384658020625a9944> /var/containers/Bundle/Application/C6BE271D-2EAC-42C0-8E72-4523F88C76B2/AppName.app/AppName
```

其中0x100040000、0x100060000、0x5000是加载地址(loadingAddress), 而arm64、armv7 是 architecture 的值(architectureValue)，这两个值后面都要用。

3. 然后执行atos命令,输入成功，进入待输入状态

```shell
xcrun atos -o appName.app.dSYM/Contents/Resources/DWARF/appName -l loadingAddress -arch architectureValue
```

4. 此时输入App对应的Crash地址，得到发生crash的信息。

**实例1** ：

```
grep "AppName arm" *crash
    xcrun atos -o AppName.app.dSYM/Contents/Resources/DWARF/AppName -l 0x100040000 -arch arm64
```

**实例2** ：

```
grep "AppName arm" *crash
    xcrun atos -o AppName.app.dSYM/Contents/Resources/DWARF/AppName -l 0x5000 -arch armv7
```
