#dis
本文简单介绍了iOS中离屏渲染的相关内容呢。

### 1.什么是离屏渲染：

要了解离屏渲染，我们先简单了解一下非离屏渲染的逻辑

#### 非离屏渲染

非离屏渲染也就是正常的渲染流程，简要流程如图：
[image:E0EF46CC-D66B-43E7-A9E9-858131BBE6D9-455-000029A84238546E/A2A975B2-1F6A-4B68-9DE5-66E8611BE25C.png]
![[A2A975B2-1F6A-4B68-9DE5-66E8611BE25C.png]]

APP将要渲染的信息提交给CPU，CPU通过一定的处理后提交给GPU。GPU不停的将内容渲染完成放到帧缓冲区中（FrameBuffer）。最后显示到屏幕上。

#### 离屏渲染

离屏渲染简要流程如图：
[image:D5FE9B4E-151D-4823-B65B-647B59DA8468-455-000029AB78A54AAF/873D1A2B-5D2D-4AE1-B83B-7094A8B29056.png]
![[873D1A2B-5D2D-4AE1-B83B-7094A8B29056.png]]

与普通流程不同的是，GPU把渲染好的的内容存放到离屏渲染缓冲区中，在离屏渲染缓冲区（OffscreenBuffer）中进一步做一些处理后，再提交到帧缓冲区（FrameBuffer）中。

### 2.离屏渲染对性能的影响？

* 和非离屏渲染模式相比多了进行额外的渲染合并，是对多个texture进行合并的过程。多了这么一步，所以说对性能要求更高一些。更容易出现掉帧的情况。
* 增加了额外的存储空间offscreen buffer，空间大小是屏幕的2.5倍

### 3.模拟器打开离屏渲染颜色标注

模拟器->Debug->Color Off-screen Rendered
[image:19558B82-5B91-42F7-AE87-89F0C67CD42B-455-000029AD510D2282/6ADC2C32-21B6-42EB-AE8B-25CD2E41A081.png]
![[6ADC2C32-21B6-42EB-AE8B-25CD2E41A081.png]]


开启后在模拟器界面中能看到使用离屏渲染的View了 图片中的例子是一个button，设置了一个背景色和背景图，对layer层设置cornerRadius和masksToBounds。

> masksToBounds=YES，如果不设置是看不到效果的。下面会具体说明原因。

### 4.为什么要用离屏渲染？

使用离屏渲染大致有一下两种情况：

1. 可以显示一些特殊效果，需要用到Offscreen buffer来保存中间状态。
2. 如果texture会多次显示到屏幕上，可以使用offscreen buffer进行提前渲染，并且保存在其中，达到复用的效果。layer中有shouldRasterize属性

#### 情况1：

一般都是系统去触发，例如对layer层相关处理：包括圆角、阴影、mask等等。iOS系统扁平化后出现的高斯模糊也是利用离屏渲染方式。

##### 圆角处理

设置圆角为什么会触发离屏渲染呢？这个要从layer的结构说起。 layer结构中包含3部分：

1. backgroundColor
2. contents
3. boarder相关信息（borderWidth和borderColor）

设置圆角代码，这个大家应该都知道：

```
view.layer.cornerRadius = 2
复制代码
```

我们先看看官方文档中cornerRadius相关说明：
[image:38231658-7FD1-425B-851C-2F05207DA5A4-455-000029B6B3362129/BBAF9A3B-F6E2-4359-9F86-5FC2511A3A82.png]
![[BBAF9A3B-F6E2-4359-9F86-5FC2511A3A82.png]]


Discussion中的说明： 设置cornerRadius超过0.0，不会影响contents，但是会影响background color和border。如果设置了masksToBounds会对content进行裁剪。 所以说出发离屏渲染的主要原因：

```
masksToBounds=YES;
复制代码
```

这就是上面模拟器开启离屏渲染模式中说明的为什么要设置masksToBounds的原因。masksToBounds需要对layer上的所有内容进行裁剪，过程中需要对中间值进行保存。所以进行了离屏渲染操作。

注意： 如果说layer图层比较简单，也是不会触发离屏渲染的。例如：UIImageView设置cornerRadius和masksToBounds是不会触发离屏渲染的，如果再对UIImageView设置背景色，则会触发。

```
self.view.backgroundColor = [UIColor grayColor]; 

UIImageView *imageView = [[UIImageView alloc] initWithImage:[UIImage imageNamed:@"1"]]; 
imageView.frame = CGRectMake(100, 100, 100, 100); 
imageView.layer.cornerRadius = 30; 
imageView.layer.masksToBounds = YES; 
[self.view addSubview:imageView]; 

UIImageView *imageView2 = [[UIImageView alloc] initWithImage:[UIImage imageNamed:@"1"]]; 
imageView2.backgroundColor = UIColor.redColor; 
imageView2.frame = CGRectMake(100, 250, 100, 100); 
imageView2.layer.cornerRadius = 30; 
imageView2.layer.masksToBounds = YES; 
[self.view addSubview:imageView2]; 

UIButton *btn = [UIButton buttonWithType:UIButtonTypeCustom]; 
[btn setBackgroundImage:[UIImage imageNamed:@"1"] forState:UIControlStateNormal]; 
btn.layer.cornerRadius = 30; 
tn.layer.masksToBounds = YES; 
btn.frame = CGRectMake(100, 400, 100, 100); 
[self.view addSubview:btn];

```
[image:946CABE3-E5A3-44E1-B6EE-E632852848B2-455-000029C4A7C4FD8D/F99DEACD-6D25-42D4-BCC2-CA5D4AFCCD6E.png]
![[F99DEACD-6D25-42D4-BCC2-CA5D4AFCCD6E.png]]

代码中有三个控件，前两个是UIImageView，最后一个是UIButton。

* 第一个UIImageView设置了图片没有设置背景色，没有触发离屏渲染。
* 第二个UIImageView设置了图片和背景色，触发了离屏渲染。
* 最后一个UIButton，设置图片没有设置背景色，触发了离屏渲染。原因是我们看到UIButton是由它的layer和UIImageView的layer混合起来的效果（UIButton有imageView），所以设置圆角的时候会触发离屏渲染。

#### 情况2：

是一种主动行为，是为了提高复用的效率。通常是设置layer的shouldRasterize属性来实现。

##### shouldRasterize 光栅化

shouldRasterize官方文档
[image:041DD950-3DE1-422D-9E99-15DE99BDA472-455-000029C63CDBF99E/3F54FB65-0F76-48CE-B28D-DF10D95BB64F.png]
![[3F54FB65-0F76-48CE-B28D-DF10D95BB64F.png]]

开启后，会将layer作为位图保存下来，下次直接与其他内容进行混合。这个保存的位置就是OffscreenBuffer中。这样下次需要再次渲染的时候，就可以直接拿来使用了。

shouldRasterize使用建议：

* layer不复用，没必要使用shouldRasterize
* layer不是静态的，也就是说要频繁的进行修改，没必要使用shouldRasterize
* 时间方面：离屏渲染缓存有100ms时间限制，超过该时间的内容会被丢弃，进而不能达到复用的目的
* 空间方面：离屏渲染空间是屏幕像素的2.5倍，如果超过也无法复用。

### 5.离屏渲染逻辑

图层的叠加绘制大致遵循“画家算法”，也就是由远到近的方式将图层绘制到屏幕上，绘制近距离图层会有覆盖远图层的逻辑。
[image:365E2DE7-D76D-40C0-8E32-3BFE60C7198F-455-000029C88656C315/488D14DC-7DE6-4F35-95A0-03330DB59699.png]
![[488D14DC-7DE6-4F35-95A0-03330DB59699.png]]

普通渲染方式，绘制完一层图层后，直接舍弃掉。紧接着绘制稍近的图层。以此类推：
[image:B31BEB1D-EE46-4A0C-9C55-87141E8615C7-455-000029C91E34EFA0/2CD86A6B-0FEB-4404-82F0-61C6FA214250.png]
![[2CD86A6B-0FEB-4404-82F0-61C6FA214250.png]]

而我们进行离屏渲染方式（圆角、剪裁等操作）时，每层图层绘制完不会马上删除，而是先保存在离屏缓存区中，等图层绘制完成后，再依次进行特殊处理（圆角、剪裁等操作）。  
[image:30B56C28-EC32-4990-B3AE-70F5B7FA4F5E-455-000029CCA04C206B/52EC43FC-793D-4525-8A11-47E39E7EE206.png]
![[52EC43FC-793D-4525-8A11-47E39E7EE206.png]]

注意：通常会触发离屏渲染，除了圆角、剪裁外，还有设置了透明度+组透明（layer.allowsGroupOpacity+layer.opacity），阴影属性（shadowOffset 等）都会产生类似的效果，因为组透明度、阴影都是和裁剪类似的，会作用与 layer 以及其所有 sublayer 上，这就导致必然会引起离屏渲染。

### 6.避免圆角离屏渲染

为了避免设置圆角时引起的离屏渲染操作，可以用一下方案代替直接设置圆角的操作

* 直接更换支援，让UI提供带圆角的图片。
* 使用layer.mask属性，增加一个和背景色相同的遮罩覆盖上层，盖住四个角，营造出圆角的形状。但这种方式难以解决背景色为图片或渐变色的情况。
* 使用贝塞尔曲线绘制闭合圆角的矩形，在上下文中设置只有内部可见，再将不带圆角的 layer 渲染成图片，添加到贝塞尔矩形中。这种方法效率更高，但是 layer 的布局一旦改变，贝塞尔曲线都需要手动地重新绘制，所以需要对 frame、color 等进行手动地监听并重绘。
* CoreGraphics重写 drawRect:，用 CoreGraphics 相关方法，在需要应用圆角时进行手动绘制。不过 CoreGraphics 效率也很有限，如果需要多次调用也会有效率问题。

### 7.触发离屏渲染原因的总结

1. 设置layer.mashsToBounds/view.clipsToBounds
2. 设置layer.mask
3. 设置layer.shadow等相关属性
4. 设置layer.shouldRasterize光栅化
5. 设置了组透明度为 YES，并且透明度不为 1 的 layer (layer.allowsGroupOpacity/layer.opacity)
6. 绘制了文字的 layer (UILabel, CATextLayer, Core Text 等)

### 8.参考文章

[iOS 渲染原理解析](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2F6ckRnyAALbCsXfZu56kTDw%3Fscene%3D25)

[iOS离屏渲染](https://juejin.cn/post/6847902220235571213)