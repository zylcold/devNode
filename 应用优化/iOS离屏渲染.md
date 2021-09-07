#display

本文简单介绍了iOS中离屏渲染的相关内容呢。

### 1.什么是离屏渲染：

要了解离屏渲染，我们先简单了解一下非离屏渲染的逻辑

#### 非离屏渲染
![[非离屏渲染]]

#### 离屏渲染
![[离屏渲染]]

### 2.离屏渲染对性能的影响？
![[离屏渲染对性能的影响？]]

### 3.模拟器打开离屏渲染颜色标注

模拟器->Debug->Color Off-screen Rendered

![[6ADC2C32-21B6-42EB-AE8B-25CD2E41A081.png]]


开启后在模拟器界面中能看到使用离屏渲染的View了 图片中的例子是一个button，设置了一个背景色和背景图，对layer层设置cornerRadius和masksToBounds。

> masksToBounds=YES，如果不设置是看不到效果的。下面会具体说明原因。

### 4.为什么要用离屏渲染？
![[为什么要用离屏渲染？]]

### 5.离屏渲染逻辑
![[离屏渲染逻辑]]

### 6.避免圆角离屏渲染
![[避免圆角离屏渲染]]


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