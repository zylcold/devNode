## 理解签名

### 基础回顾

非对称加密。在密码学中，非对称加密需要两个密钥：公钥和私钥。私钥加密的只能用公钥解密，公钥加密的只能用私钥解密。

数字签名。数字签名表示我对数据做了个标记，表示这是我的数据，没有经过篡改。

数据发送方Leo产生一对公私钥，私钥自己保存，公钥发给接收方Lina。Leo用摘要算法，对发送的数据生成一段摘要，摘要算法保证了只要数据修改，那么摘要一定改变。然后用私钥对这个摘要进行加密，和数据一起发送给Lina。

[image:7E6909D4-7FA2-42AC-8CB1-8F81B1F716D3-1690-000006944B72E94D/image_2.png]
![[image_2.png]]

Lina收到数据后，用公钥解密签名，得到Leo发过来的摘要；然后自己按照同样的摘要算法计算摘要，如果计算的结果和Leo的一样，说明数据没有被篡改过。

[image:370E6500-E4E3-4AAF-ACE1-9249BED656F3-1690-000006944B50BF8D/image_16.png]
![[image_16.png]]

但是，现在还有个问题：Lina有一个公钥，假如攻击者把Lina的公钥替换成自己的公钥，那么攻击者就可以伪装成Leo进行通信，所以 **Lina需要确保这个公钥来自于Leo** ，可以通过数字证书来解决这个问题。

> 数字证书由CA（Certificate Authority）颁发，以Leo的证书为例，里面包含了以下数据： **签发者** ； **Leo的公钥** ； **Leo使用的Hash算法** ； **证书的数字签名** ；到期时间等。

有了数字证书后，Leo再发送数据的时候，把自己从CA申请的证书一起发送给Lina。Lina收到数据后，先用CA的公钥验证证书的数字签名是否正确，如果正确说明证书没有被篡改过，然后以信任链的方式判断是否信任这个证书，如果信任证书，取出证书中的数据，可以判断出证书是属于Leo的，最后从证书中取出公钥来做数据签名验证。

### iOS App签名

为什么要对App进行签名呢？ **签名能够让iOS识别出是谁签名了App，并且签名后App没有被篡改过** ，

除此之外，Apple要严格控制App的分发：

1. App来自Apple信任的开发者
2. 安装的设备是Apple允许的设备
#### 证书

通过上文的讲解，我们知道数字证书里包含着申请证书设备的公钥，所以在Apple开发者后台创建证书的时候，需要上传CSR文件(Certificate Signing Request)，用keychain生成这个文件的时候，就生成了一对公/私钥： **公钥在CSR里，私钥在本地的Mac上** 。Apple本身也有一对公钥和私钥： **私钥保存在Apple后台，公钥在每一台iOS设备上** 。

[image:F3C6C0CA-6824-40BF-B4CE-A0AEE32045D9-1690-000006944B350FC7/image_12.png]
![[image_12.png]]

#### Provisioning Profile

iOS App安装到设备的途径(非越狱)有以下几种：

1. 开发包(插线，或者archive导出develop包)
2. Ad Hoc
3. App Store
4. 企业证书
开发包和Ad Hoc都会严格限制安装设备，为了把设备uuid等信息一起打包进App，开发者需要配置Provisioning Profile。

[image:668F3A1D-AF1F-4180-8B16-B436CCB4589B-1690-000006944B14BC4E/image_14.png]
![[image_14.png]]

可以通过以下命令来查看Provisioning Profile中的内容：

```
security cms -D -i embedded.mobileprovision > result.plist
open result.plist
12
```

本质上就是一个编码过后的plist

[image:E03A2F06-C7C7-43B1-9F79-40A8EEFFD350-1690-000006944AF6BEF8/image_10.png]

![[image_10.png]]
#### iOS签名

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

[image:71D9B1CE-FBDB-4EA8-A464-2360D93F9F82-1690-000006944ADC00DE/image_5.png]
![[image_5.png]]

#### 验证

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