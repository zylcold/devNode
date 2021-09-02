# IPA包的内容

例如，我们通过iTunes Store下载微信，然后获得ipa安装包，然后实际看看其安装包的内容。

![[20161122233433923.jpeg]]

* 右键ipa，重命名为 `.zip`
* 双击zip文件，解压缩后会得到一个文件夹。所以，ipa包就是一个普通的压缩包。

![[20161122233734162.jpeg]]

- 右键图中的 ` WeChat ` ，选择显示包内容，然后就能够看到实际的ipa包内容了。

---

# 二进制文件的内容

通过XCode的Link Map File，我们可以窥探二进制文件中布局。
在XCode -> Build Settings -> 搜索map -> 开启Write Link Map File

![[20161203212636392.jpeg]]

开启后，在编译，我们可以在对应的Debug/Release目录下看到对应的link map的text文件。
默认的目录在

```shell
~/Library/Developer/Xcode/DerivedData/<TARGET-NAME>-对应ID/Build/Intermediates/<TARGET-NAME>.build/Debug-iphoneos/<TARGET-NAME>.build/
```

例如，我的TargetName是 `EPlusPan4Phone` ，目录如下

```shell
/Users/huangwenchen/Library/Developer/Xcode/DerivedData/EPlusPan4Phone-eznmxzawtlhpmadnbyhafnpqpizo/Build/Intermediates/EPlusPan4Phone.build/Debug-iphonesimulator/EPlusPan4Phone.build
```

这个映射文件的主要包含以下部分：

## Object files

这个部分包括的内容

.o 文文件，也就是上文提到的.m文件编译后的结果。
.a文件
需要link的framework

```
＃！ Arch: x86_64
＃Object files:
[0]linker synthesized
[1]/EPlusPan4Phone.build/EPlusPan4Phone.app.xcent
[2]/EPlusPan4Phone.build/Objects-normal/x86_64/ULWBigResponseButton.o
…
[1175]/UMSocial_Sdk_4.4/libUMSocial_Sdk_4.4.a(UMSocialJob.o)
[1188]/iPhoneSimulator10.1.sdk/System/Library/Frameworks//Foundation.framework/Foundation
```

这个区域的存储内容比较简单：前面是文件的编号，后面是文件的路径。文件的编号在后续会用到

## Sections

这个区域提供了各个段（Segment）和节（Section）在可执行文件中的位置和大小。这个区域完整的描述克可执行文件中的全部内容。

其中，段分为两种

__TEXT 代码段
__DATA 数据段

例如，之前写的一个App，Sections区域如下，可以看到，代码段的

__text节的地址是0x1000021B0，大小是0x0077EBC3，而二者相加的下一个位置正好是__stubs的位置0x100780D74。

```
# Sections:
# 位置       大小        段       节
# Address	Size    	Segment	Section
0x1000021B0	0x0077EBC3	__TEXT	__text //代码
0x100780D74	0x00000FD8	__TEXT	__stubs
0x100781D4C	0x00001A50	__TEXT	__stub_helper
0x1007837A0	0x0001AD78	__TEXT	__const //常量
0x10079E518	0x00041EF7	__TEXT	__objc_methname //OC 方法名
0x1007E040F	0x00006E34	__TEXT	__objc_classname //OC 类名
0x1007E7243	0x00010498	__TEXT	__objc_methtype  //OC 方法类型
0x1007F76DC	0x0000E760	__TEXT	__gcc_except_tab 
0x100805E40	0x00071693	__TEXT	__cstring  //字符串
0x1008774D4	0x00004A9A	__TEXT	__ustring  
0x10087BF6E	0x00000149	__TEXT	__entitlements 
0x10087C0B8	0x0000D56C	__TEXT	__unwind_info 
0x100889628	0x000129C0	__TEXT	__eh_frame
0x10089C000	0x00000010	__DATA	__nl_symbol_ptr
0x10089C010	0x000012C8	__DATA	__got
0x10089D2D8	0x00001520	__DATA	__la_symbol_ptr
0x10089E7F8	0x00000038	__DATA	__mod_init_func
0x10089E840	0x0003E140	__DATA	__const //常量
0x1008DC980	0x0002D840	__DATA	__cfstring
0x10090A1C0	0x000022D8	__DATA	__objc_classlist // OC 方法列表
0x10090C498	0x00000010	__DATA	__objc_nlclslist 
0x10090C4A8	0x00000218	__DATA	__objc_catlist
0x10090C6C0	0x00000008	__DATA	__objc_nlcatlist
0x10090C6C8	0x00000510	__DATA	__objc_protolist // OC协议列表
0x10090CBD8	0x00000008	__DATA	__objc_imageinfo
0x10090CBE0	0x00129280	__DATA	__objc_const // OC 常量
0x100A35E60	0x00010908	__DATA	__objc_selrefs
0x100A46768	0x00000038	__DATA	__objc_protorefs 
0x100A467A0	0x000020E8	__DATA	__objc_classrefs 
0x100A48888	0x000019C0	__DATA	__objc_superrefs // OC 父类引用
0x100A4A248	0x0000A500	__DATA	__objc_ivar // OC iar
0x100A54748	0x00015CC0	__DATA	__objc_data
0x100A6A420	0x00007A30	__DATA	__data
0x100A71E60	0x0005AF70	__DATA	__bss
0x100ACCDE0	0x00053A4C	__DATA	__common
```

## Symbols

Section部分将二进制文件进行了一级划分。而，Symbols对Section中的各个段进行了二级划分，
例如，对于 `__TEXT __text`,表示代码段中的代码内容。

```
0x1000021B0	0x0077EBC3	__TEXT	__text //代码
```

而对应的 `Symbols` ，起始地址也是 `0x1000021B0` 。其中，文件编号和上文的编号对应

```
[2]/EPlusPan4Phone.build/Objects-normal/x86_64/ULWBigResponseButton.o
```

具体内容如下

```
# Symbols:
  地址     大小          文件编号    方法名
# Address	Size    	File       Name
0x1000021B0	0x00000109	[  2]     -[ULWBigResponseButton pointInside:withEvent:]
0x1000022C0	0x00000080	[  3]     -[ULWCategoryController liveAPI]
0x100002340	0x00000080	[  3]     -[ULWCategoryController categories]
....

```

到这里，我们知道OC的方法是如何存储的，我们再来看看ivar是如何存储的。
首先找到数据栈中 `__DATA __objc_ivar`

```
0x100A4A248	0x0000A500	__DATA	__objc_ivar
```

然后，搜索这个地址 `0x100A4A248` ，就能找到ivar的存储区域。

```
0x100A4A248	0x00000008	[  3] _OBJC_IVAR_$_ULWCategoryController._liveAPI
```

值得一提的是，对于String，会显式的存储到数据段中，例如,

```
0x1008065C2	0x00000029	[ 11] literal string: http://sns.whalecloud.com/sina2/callback
```

所以，若果你的加密Key以明文的形式写在文件里，是一件很危险的事情。