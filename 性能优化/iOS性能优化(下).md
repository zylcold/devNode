该篇延续上篇关于iOS性能优化剩余的两点做讲解：
4 优化建议（启动、界面、耗能等）
[[应用瘦身]]

# 4. 优化建议（启动、界面、耗能等）
启动相关知识 启动分为2种： 冷启动和热启动 启动的时间分为两部分：main函数执行之前、main函数至应用启动完成
### 4.1 启动优化建议
### 4.1 main函数之前
减少动态库、合并一些动态库 减少Objc类、分类的数量、减少Selector数量
### 4.2 main函数至应用启动完成
耗时操作，不要放在finishLaunching方法中, 初始化，viewdidload等初始显示界面的生命周期函数中
动态库对启动时间的影响测试  [www.cocoachina.com/ios/2016112…](http://www.cocoachina.com/ios/20161125/18179.html)  使用了dynamic framework后会增加app启动时间。如果你的数量在25个左右，相比OC的静态framework启动时间会增加0.5s左右。我个人对于iOS提高加载framework的时间不太抱有希望，苹果让自定义的framework常驻内存似乎也无望，这个时间短期内可能无法抹平。
如果人为的把一些外部依赖手动管理在一个framework也是可行，但是如果复杂一点包的互相依赖的情况会比较费心。
如果公司对性能有着苛刻要求可能500ms是难以忍受的。但是我觉得对于大多数产品而言，牺牲这500ms的性能相比于使用OC，我觉得还是用OC比较难受。
关于4.1 main函数之前的启动时间测量方法: DYLD_PRINT_STATISTICS 设置为 1

[image:214241C9-2E6F-4473-A36C-9AC68CE91640-439-0000240F7333C29A/6e75971bd1f94827a36cd040621d5e09~tplv-k3u1fbpfcp-watermark.image.png]
![[6e75971bd1f94827a36cd040621d5e09~tplv-k3u1fbpfcp-watermark.image.png]]

[image:C270C177-F495-4613-B0C6-7FEFD253A86F-439-0000240F72FB341B/ee5117eb14744966963658374c618eac~tplv-k3u1fbpfcp-watermark.image.png]
![[ee5117eb14744966963658374c618eac~tplv-k3u1fbpfcp-watermark.image.png]]

### 4.2 界面优化
### 4.2.1 卡顿的原理
掉帧
[image:0F2CF84B-A33E-4CC4-8F55-FE4F93D77C59-439-0000240F72C53819/22763d55f4c1486094d7289953389dd8~tplv-k3u1fbpfcp-watermark.image.png]
![[22763d55f4c1486094d7289953389dd8~tplv-k3u1fbpfcp-watermark.image.png]]

### 4.2.2 界面流畅度的评测
**界面优化建议(后面我会单独写一篇专门讲解界面优化)**
耗时操作，不要放在主线程
合理使用CPU与GPU
复制代码
CPU: 计算显示内容， 比如视图的创建、布局计算、图片解码、文本绘制. 这些一般都是uikit的问题,苹果是用autolayout布局的， GPU :会把CPU计算好的数据进行渲染，视图+数据+帧率 = 很多图片
[摘自]  [www.javashuo.com/article/p-y…](http://www.javashuo.com/article/p-yjmjlubo-bz.html) 
**4.2.2.1 Color Blended Layers (图层混合)**
设置opaque 属性为true。 所有控件默认都为true可以忽略 给View设置一个不透明的颜色，没有特殊须要设置白色便可。 尤其是label.backgroudColor
label.backgroundColor = [UIColor whiteColor];
label.layer.masksToBounds = YES;
复制代码
到这里你可能奇怪，设置label的背景色第一行不就够了么，为何还有第二行？这是由于若是label的内容是中文，label实际渲染区域要大于label的size，最外层多了一个sublayer，若是不设置第二行label的边缘外层灰出现图层混合的红色，所以须要在label内容是中文的状况下加第二句。单独使用label.layer.masksToBounds = YES是不会发生离屏渲染。 注意点：UIImageView控件比较特殊，不只须要自身这个容器是不透明的，而且imageView包含的内容图片也必须是不透明的，若是你本身的图片出现了图层混合红色，先检查是否是本身的代码有问题，若是确认代码没问题，就是图片自身的问题
**4.2.2.2 光栅化**
适用状况：通常在图像内容不变的状况下才使用光栅化，例如设置阴影耗费资源比较多的静态内容，若是使用光栅化对性能的提高有必定帮助。 非适用状况：若是内容会常常变更,这个时候不要开启,不然会形成性能的浪费。 例如咱们在使用tableViewCell中，通常不要用光栅化，由于tableViewCell的绘制很是频繁，内容在不断的变化，若是使用了光栅化，会形成大量的离屏渲染下降性能。
**4.2.2.3 Offscreen rendering (圆角)**
阴影绘制shadow:使用ShadowPath来替代shadowOffset等属性的设置 imageViewLayer.shadowPath = CGPathCreateWithRect(imageRect, NULL); 利用GraphicsContex生成一张带圆角的图片或者view，这里不写具体实现过程，须要的能够度娘Copy，不少现成的代码。
### 4.3卡顿检测:
FPS 50-60帧是非常流畅的，低于高于都不正常。YYFPSLabel
```objc
//YYLabel YYDemo
YYFPSLabel *_fpsLabel = [YYFPSLabel new];
[_fpsLabel sizeToFit];
_fpsLabel.bottom = KScreenHeight - 55;
_fpsLabel.right = KScreenWidth - 10;
//    _fpsLabel.alpha = 0;
[kAppWindow addSubview:_fpsLabel];

```
如果界面比较复杂的，建议使用AsyncDisplayKit做控件绘制，而不是苹果自带的autolayout。
