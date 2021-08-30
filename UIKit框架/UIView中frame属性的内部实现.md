#uikit #layout

## frame、center、bounds、transform

UIView中用于表征视图在父视图中显示出来的位置和尺寸的属性是frame。同时系统还提供另外两个属性center和bounds。
其中center属性值描述视图的中心点在父视图中的位置，而bounds属性的size部分则描述视图本身固有的尺寸。

需要注意的是bounds属性中的origin部分描述的是视图内部坐标系中原点的位置，它影响着里面子视图的位置。

除此之外，系统还提供一个transform属性来实现视图的仿射变换: 比如平移、缩放、旋转、倾斜的效果。

在这四个属性中，除了frame属性是计算属性外，其他三个属性都是实体属性。frame的值是依赖这三个属性计算出来的。

在介绍frame的计算公式前先要了解三个概念：图层(CALayer)、锚点(Anchor Point)、仿射变换。

> iOS和macOS两个系统的参考坐标系不一致，对于iOS来说原点默认在视图的左上角位置，而对于macOS来说原点默认是在视图的左下角位置。

## UIView和CALayer的定位映射关系

UIView是对视图的抽象类，它主要用来负责数据的存储和操作逻辑的实现。
而CALayer则是对视图在屏幕上的渲染和显示信息的抽象类。

对于一个视图的位置和尺寸来说也是属于渲染显示信息的一部分。

因此上述视图中的几个属性的内部实现其实是委托给CALayer中的对应属性来实现的，其对应关系表如下：



-- | --
UIView | CALayer  

frame  frame   
center position   
bounds bounds   
transform affineTransform

### 锚点(Anchor Point)

所谓锚点就是用来确定视图在父视图中的位置而在视图内某个点的相对坐标值。视图是一个矩形区域，里面有无数个点，只要明确了视图内某个点的坐标值在父视图中的位置，那么这个视图的位置就可以被确认，而这个被指定的视图内的位置坐标点就是锚点。系统为了表征锚点而为层提供了一个anchorPoint属性。锚点是一个相对坐标值，其左上角的位置是(0,0)而右下角的位置是(1,1)中心点的锚点值就是(0.5,0.5)了*(对于macOS系统来说，因为坐标系的不同，(0,0)位置位于左下角，而(1,1)位置则位于右上角)*。默认情况下系统将层内的中心点作为锚点，这也就是视图的center属性描述的是视图的中心点在父视图的位置的原因。锚点是CALayer中的概念，而不是视图的概念。就如上面的视图属性和层属性的对应关系可以看出来视图的center属性对应的是层的position属性。其实后者更能表现锚点位置这个概念，因为position表明的是层的锚点在父层中的绝对位置。虽然默认情况下锚点是(0.5,0.5)而这个设定刚好和center属性所表明的意思是一致的，但是我们是可以改变锚点的值的。就比如下面的代码：

```objc
//建立时A的frame是(0,0,100,100), 而center值是(50,50)表明center刚好是中心点位置。
UIView *A = [[UIView alloc] initWithFrame:CGRectMake(0,0,100,100)];
A.anchorPoint = CGPointMake(0,0);

//这时候frame的值将变为(50,50,100,100), 但是center的值还是(50,50)却不是表明视图的中心点位置了。
CGRect frame = A.frame;  

```

### 仿射变换

所谓仿射变换就是对一个坐标空间的所有点进行一次线性变换并接上一个平移处理。iOS系统中的视图的属性transform就是用来实现对视图进行仿射变换处理的。通过仿射变换我们可以很轻易的实现对视图的移动、缩放、旋转、倾斜等处理。transform属性是一个结构体类型的数据：

```objc
struct CGAffineTransform {
  CGFloat a, b, c, d;
  CGFloat tx, ty;
};

```

下面的公式就是利用这个结构体来实现坐标点由(x0,y0)到(x1,y1)的仿射变换处理：

```objc
x1 = a*x0 + b*y0 + tx
y1 = c*x0 + d*y0 + ty
```

系统提供了众多以CGAffine开头的函数API来构造和处理各种常见的仿射变换操作。默认情况下视图的transform属性值是一个 **CGAffineTransformIdentity** 表明不会对视图进行任何仿射变换处理。

一个视图最终渲染到屏幕上的位置和尺寸是由视图的原始位置和尺寸外加仿射变换来决定的。视图渲染到屏幕上的最终位置和尺寸可以通过frame属性来获取。

### frame的计算规则

从上面的介绍中可以看出，一个视图最终渲染出来的位置和尺寸需要通过设置视图或者层的center、bounds、transform、anchorPoint四个属性来完成。但是这样太过于麻烦，因此为了简化操作可以通过frame属性来完成这些设置。 frame属性是一个计算属性。下面就是这个属性的获取和设置的实现的伪代码：

```objc
-(CGRect)frame
{
      CGRect retValue = CGRectZero;
      if (CGAffineTransformIsIdentity(self.transform))
      { //没有设置仿射变换的情况下
           
            //位置等于中心点的位置减去视图尺寸乘以锚点的值。
            retValue.origin.x = self.center.x - self.bounds.size.width * self.layer.anchorPoint.x;
            retValue.origin.y = self.center.y - self.bounds.size.height * self.layer.anchorPoint.y;
           //尺寸等于视图的尺寸
            retValue.size.width = self.bounds.size.width;
            retValue.size.height = self.bounds.size.height;
      }
      else
      {
            CGAffineTransform left =  CGAffineTransformMakeTranslation(-1 * self.bounds.size.width * self.layer.anchorPoint.x, -1 * self.bounds.size.height * self.layer.anchorPoint.y);
            //因为下面的坐标变换应用是从(0,0)开始的，因此这里的right指定中心点的位置，也就是下面的复合变换右乘right来实现位置的变换处理。
            CGAffineTransform right =  CGAffineTransformMakeTranslation(self.center.x, self.center.y);
            //整个复合变换是left 左乘 视图的tansform属性然后再右乘right变换。
            CGAffineTransform concat  = CGAffineTransformConcat(CGAffineTransformConcat(left, self.transform), right); 
            retValue  = CGRectApplyAffineTransform(CGRectMake(0,0,self.bounds.size.width, self.bounds.size.height), concat);
      }

    return retValue
}

-(void)setFrame:(CGRect)frame
{
      //设置frame时并没有考虑到仿射坐标变换属性transform。
      self.center.x = frame.origin.x + self.bounds.size.width * self.layer.anchorPoint.x;
      self.center.y = frame.origin.y  + self.bounds.size.height * self.layer.anchorPoint.y;
      self.bounds.size.width = frame.size.width;
      self.bounds.size.height = frame.size.height;
}

```

从上面的代码可以看出： **当一个视图设置了非CGAffineTransformIdentity的仿射变化值后，我们不能再通过设置frame属性的值来修改视图的位置和尺寸了，否则最终展示的效果未可知。** 因此当对视图设置了仿射变换属性后，如果需要调整视图的位置和尺寸时我们需要操作的是center属性和bounds属性而不能在操作frame属性了。比如下面的例子：

```objc
//假设视图尺寸由原来的(width0, height0)改变为(width1, height1)那么正确代码应该如下设置：
//尺寸直接设置。
view.bounds.size = CGSizeMake(width1, height1);
//位置需要调整
view.center.x += (width1 - width0) * view.layer.anchorPoint.x;
view.center.y += (height1 - height0) * view.layer.anchorPoint.y; 

//假设视图位置由原来的(x0,y0)改为(x1,y1)那么正确的代码应该如下设置：
view.center.x = x1 + view.bounds.size.width * view.layer.anchorPoint.x;
view.center.y = y1 + view.bounds.size.height * view.layer.anchorPoint.y;

```

> AutoLayout在完成布局后，所计算出来的位置和尺寸内部修改的值是center和bounds两个属性，因此最终的展示效果不会因为仿射变换而产生异常。同时这也解释了为什么通过AutoLayout设置约束后修改frame属性来改变位置和尺寸不会起作用的原因。

> [MyLayout](https://link.juejin.im/?target=https://github.com/youngsoft/MyLinearLayout) 布局计算早期是通过修改视图的frame属性来完成布局的，但是后来发现有程序员在设置了仿射变换属性后发现视图展示出现异常，后来的版本内部也统一改为了修改视图的center和bounds属性来解决这类问题。
