#build #sign

# 理解签名

## 基础回顾

![[非对称加密]]

![[数字签名]]

有了[[数字证书]]后，Leo再发送数据的时候，把自己从CA申请的证书一起发送给Lina。Lina收到数据后，先用CA的公钥验证证书的数字签名是否正确，如果正确说明证书没有被篡改过，然后以信任链的方式判断是否信任这个证书，如果信任证书，取出证书中的数据，可以判断出证书是属于Leo的，最后从证书中取出公钥来做数据签名验证。

## iOS App签名

为什么要对App进行签名呢？ **签名能够让iOS识别出是谁签名了App，并且签名后App没有被篡改过** 

除此之外，Apple要严格控制App的分发：

1. App来自Apple信任的开发者
2. 安装的设备是Apple允许的设备


### 证书

通过上文的讲解，我们知道数字证书里包含着申请证书设备的公钥，所以在Apple开发者后台创建证书的时候，需要上传CSR文件(Certificate Signing Request)，用keychain生成这个文件的时候，就生成了一对公/私钥： **公钥在CSR里，私钥在本地的Mac上** 。Apple本身也有一对公钥和私钥： **私钥保存在Apple后台，公钥在每一台iOS设备上** 。

![[image_12.png]]

### Provisioning Profile

iOS App安装到设备的途径(非越狱)有以下几种：

1. 开发包(插线，或者archive导出develop包)
2. Ad Hoc
3. App Store
4. 企业证书


开发包和Ad Hoc都会严格限制安装设备，为了把设备uuid等信息一起打包进App，开发者需要配置Provisioning Profile。

![[image_14.png]]

可以通过以下命令来查看Provisioning Profile中的内容：

```shell
security cms -D -i embedded.mobileprovision > result.plist
open result.plist
```

本质上就是一个编码过后的plist
![[image_10.png]]
### iOS签名

生成安装包的最后一步，XCode会调用 `codesign` 对Product.app进行签名。
创建一个额外的目录 `_CodeSignature` 以plist的方式存放安装包内每一个文件签名

```xml

<key>Base.lproj/LaunchScreen.storyboardc/01J-lp-oVM-view-Ze5-6b-2t3.nib</key>
<data>
T2g5jlq7EVFHNzL/ip3fSoXKoOI=
</data>
<key>Info.plist</key>
<data>
5aVg/3m4y30m+GSB8LkZNNU3mug=
</data>
<key>PkgInfo</key>
<data>
n57qDP4tZfLD1rCS43W0B4LQjzE=
</data>
<key>embedded.mobileprovision</key>
<data>
tm/I1g+0u2Cx9qrPJeC0zgyuVUE=
</data>
...


```

代码签名会直接写入到mach-o的可执行文件里，值得注意的是签名是以页(Page)为单位的，而不是整个文件签名：

![[image_5.png]]

### 验证

在安装App的时候，

* 从embedded.mobileprovision取出证书，验证证书是否来自Apple信任的开发者
* 证书验证通过后，从证书中取出Leo的公钥
* 读取 `_CodeSignature` 中的签名结果，用Leo的公钥验证每个文件的签名是否正确
* 文件 `embedded.mobileprovision` 验证通过后，读取里面的设备id列表，判断当前设备是否可安装(App Store和企业证书不做这步验证)
* 验证通过后，安装App

启动App的时候：

* 验证bundle id，entitlements和 `embedded.mobileprovision` 中的AppId，entitlements是否一致
* 判断device id包含在embedded.mobileprovision里
	* App Store和企业证书不做验证
	* 如果是企业证书，验证用户是否信任企业证书
* App启动后，当缺页中断(page fault)发生的时候，系统会把对应的mach-o页读入物理内存，然后验证这个page的签名是否正确。
* 以上都验证通过，App才能正常启动