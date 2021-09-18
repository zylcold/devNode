#uikit #layout #autolayout

[原文链接](http://chuquan.me/2019/09/25/systematic-understand-ios-autolayout/)


最近准备阅读 Masonry 的源代码，学习一下其中的设计思想。然而，阅读了一部分之后，发现自己对 iOS 自动布局了解的不够系统，也不够深入。于是，准备好好学习学习 iOS 自动布局的基础知识。

下面是我对 iOS 布局系统的一些整理和总结，当然，自动布局是其中的重点。

# 概述

苹果在 iPhone 4 时推出了绝对布局，随着 iOS 设备不断增多，苹果在 iOS 6 时又推出了自动布局（Auto Layout）。在自动布局逐步完善的过程中，苹果也推出了诸如：Size Class、Stack View、UILayoutGuide 等技术，但是它们的本质都是基于自动布局。


历史：
iOS 5 >  绝对布局
iOS 6 > Auto Layout、VFL
iOS 9 > UILayoutGuide、NSLayoutAnchor、UIStackView
iOS 11 > Safe Area 


# 来源

Cassorwary算法

![[Cassorwary算法]]


2011 年，苹果将 Cassowary 算法应用到了自家的布局引擎 Auto Layout 中。

# 约束

Cassowary 的核心是基于 **约束（Constraint）** 来描述视图之间的关系。约束本质上就是一个方程式：

```
item1.attribute1 = multiplier × item2.attribute2 + constant
```

下面我们通过一个简单的约束来介绍约束方程式。

![[16ea0935049caa0b.jpeg]]


该约束表示红色视图的左边界在蓝色视图的右边界再往右 8 个像素点。 **注意，这里的 `=`并不是赋值的意思，而是相等的意思** 。

在自动布局系统中，约束不仅可以定义两个视图之间的关系，还可以定义单个视图的两个不同属性之间的关系，如：在视图的高度和宽度之间设置比例。 **一般而言，一个视图需要四个约束来决定其大小和位置** 。

## 约束规则

上述约束方程式主要描述了两个视图属性之间的关系。那么，我们来看一下 iOS 定义了哪些属性和关系。

### 属性

苹果使用 `NSLayoutAttribute` 类型的枚举值来表示布局属性，其主要包含以下这些属性：

```objc
typedef NS_ENUM(NSInteger, NSLayoutAttribute) {
    // 视图位置
    NSLayoutAttributeLeft = 1,
    NSLayoutAttributeRight,
    NSLayoutAttributeTop,
    NSLayoutAttributeBottom,
    // 视图前后
    NSLayoutAttributeLeading,
    NSLayoutAttributeTrailing,
    // 视图宽高
    NSLayoutAttributeWidth,
    NSLayoutAttributeHeight,
    // 视图中心
    NSLayoutAttributeCenterX,
    NSLayoutAttributeCenterY,
    // 视图基线
    NSLayoutAttributeLastBaseline,
    NSLayoutAttributeFirstBaseline NS_ENUM_AVAILABLE_IOS(8_0),
    
    NSLayoutAttributeLeftMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeRightMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeTopMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeBottomMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeLeadingMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeTrailingMargin NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeCenterXWithinMargins NS_ENUM_AVAILABLE_IOS(8_0),
    NSLayoutAttributeCenterYWithinMargins NS_ENUM_AVAILABLE_IOS(8_0),
    
    // 占位符，在与另一个约束的关系中没有用到某个属性时可以使用占位符
    NSLayoutAttributeNotAnAttribute = 0
};
```

值得注意的是， `NSLayoutAttribute` 有类似 `NSLayoutAttributeLeft` 和 `NSLayoutAttributeLeftMargin` 这样的枚举。两者的区别是：

* `NSLayoutAttributeLeft` 表示视图的最左边；
* `NSLayoutAttributeLeftMargin` 表示视图的左边，距离最左边有多大的 margin 与视图的 `layoutMargins` 有关。

关于 `layoutMargins` 我们会在下文提到。

![[16ea0935052b75d9.jpeg]]

### 关系

苹果使用 `NSLayoutRelation` 类型的枚举值来表示属性关系，其主要包含以下这些关系：

```objc
typedef NS_ENUM(NSInteger, NSLayoutRelation) {
    NSLayoutRelationLessThanOrEqual = -1,
    NSLayoutRelationEqual = 0,
    NSLayoutRelationGreaterThanOrEqual = 1,
};
```

## 约束层级

约束描述两个视图之间的关系，但是前提是：两个视图必须属于同一个视图层级结构。 这种层级结构有两种：

1. 一个视图是另一个视图的视图
2. 两个视图在一个窗口下有一个非 `nil` 的公共祖先视图。

![[16ea09350704231a.jpeg]]

## 约束优先级

约束具有优先级。当布局引擎计算布局时，会按照优先级从高到低的顺序逐个计算。如果发现一个可选的约束无法被满足时，就会跳过这个约束，计算下一个约束。有时候，即使一个约束无法被正好适配，它依然可以影响布局。

苹果默认定义了 4 种优先级枚举值。除此之外，苹果允许创建其他的优先级，但是其范围必须在 1~1000 之间。

```objc
static const UILayoutPriority UILayoutPriorityRequired = 1000; 
static const UILayoutPriority UILayoutPriorityDefaultHigh = 750; 
static const UILayoutPriority UILayoutPriorityDefaultLow = 250; 
static const UILayoutPriority UILayoutPriorityFittingSizeLevel = 50; 
```

# 约束创建

关于约束的创建，苹果提供了 Interface Build，可以实现以非编程的方式创建约束。但是在大型项目中，我们主要还是以编程的方式创建约束。

以编程方式创建约束的方式主要有三种：

* **约束构造器（NSLayoutConstraint）**
* **布局锚点（Layout Anchors）**
* **可视化格式语言（Visual Format Language, VFL）**

下面我们依次进行介绍。

## NSLayoutConstraint

苹果使用 `NSLayoutConstraint` 类型表示约束。 `NSLayoutConstraint` 类提供了一个构造方法可以直接创建约束。构造方法的各个参数对应着约束方程式的各个项。

```objc
+ (instancetype)constraintWithItem:(id)view1 
                         attribute:(NSLayoutAttribute)attr1 
                         relatedBy:(NSLayoutRelation)relation 
                            toItem:(id)view2 
                         attribute:(NSLayoutAttribute)attr2 
                        multiplier:(CGFloat)multiplier 
                          constant:(CGFloat)c;

```

## Layout Anchors

苹果使用 `NSLayoutAnchor` 类型表示布局锚点。在介绍 `NSLayoutAnchor` 之前，我们需要介绍一些其他的概念，如： `UILayoutGuide` （使用 `NSLayoutAnchor` 时会用到）。

### UILayoutGuide

`UILayoutGuide` 是一个虚拟的矩形区域，可以认为是一个透明的 `UIView` ，但是它不会被添加到视图层级，也不会拦截消息调用，它只是用来与 Auto Layout 交互。比如：3 个 `UIView` 排一行，相互之间的间隔相同。那么中间的间隔就可以用 `UILayoutGuide` 代替。

在了解 `UILayoutGuide` 之后，我们需要了解 `UIView` 的两个属性（ `UILayoutGuide` 类型的实例）：

* `layoutMarginsGuide` ：首次出现于 iOS 9
* `safeAreaLayoutGuide` ：首次出现于 iOS 11

#### layoutMarginsGuide

`UIView` 有一个 `UIEdgeInsets` 类型的属性 `layoutMargins` ，它表示一个视图的内容和它四个边界之间的空隙，如下图所示。

![[16ea093506c7ab2d.jpeg]]

`UIView` 的 `layoutMarginsGuide` 属性其实是 `layoutMargins` 的另一种表现形式，可用于创建布局约束。 `layoutMarginsGuide` 是一个 **只读** 属性。

#### safeAreaLayoutGuide

在 iOS 11 时，苹果提出了 Safe Area 的概念。因为 iOS 11 搭载的 iPhone X 取消了 Home 键，要为操作保留一些空间，这正好把原来的 Navigation Bar, Status Bar, Tab Bar 包含在里面。 `safeAreaLayoutGuide` 属性正是伴随 Safe Area 出现的。

`safeAreaLayoutGuide` 属性和 `layoutMarginsGuide` 一样，也是 **只读** 属性，因为它们默认都已经设定了一个虚拟区域，我们可以直接基于此区域设置约束。

![[16ea093507441c85.jpeg]]

### NSLayoutAnchor

在初步了解 `UILayoutGuide` 之后，我们再来看它所包含的成员。可以发现， `UILayoutGuide` 内部定义了一系列 `NSLayoutAnchor` 类型的成员。

事实上， `NSLayoutAnchor` 类可以通过一系列 API，创建 `NSLayoutConstraint` 类型的约束对象，来进行布局约束的设置，而不用 **直接** 和 `NSLayoutConstraint` 对象打交道。

通常，我们不会直接使用 `NSLayoutAnchor` ，而是使用它的三个子类，如下：

* `NSLayoutXAxisAnchor` ：X 轴方向的锚点，用来创建水平方向的约束
* `NSLayoutYAxisAnchor` ：Y 轴方向的锚点，用来创建垂直方向的约束
* `NSLayoutDimension` ：尺寸相关的锚点，用来创建尺寸相关的约束

`UILayoutGuide` 的诸多锚点属性可以归纳为上述三种子类中的一种。

X 轴方向的锚点属性 Y 轴方向的锚点属性 尺寸相关的锚点属性     `centerXAnchor` `centerYAnchor` `widthAnchor`   `leftAnchor` `topAnchor` `heightAnchor`   `rightAnchor` `bottomAnchor`    `leadingAnchor` `firstBaselineAnchor`    `trailingAnchor` `lasBaselineAnchor`     

> **注意** ： `leadingAnchor` 和 `leftAnchor`、`trailingAnchor` 和 `rightAnchor` 在大多数情况下效果是一样的，但还是存在本质区别： `leadingAnchor` 表示视图最前面的边界锚点，如果在英文等阅读顺序从左向右的国家，leading 就表示 left，但在阿拉伯语等阅读顺序从右向左的国家，leading 就表示 right。

一个视图那么多的锚点属性（X 轴，Y 轴，尺寸）能够和另外一个视图对应的位置锚点和尺寸锚点交互，从而确定视图的位置和尺寸。视图之间的锚点关系可以通过 API 调用来创建约束。 **前提是：只有相同子类的锚点属性之间才能交互** 。即：

* X 轴方向的锚点属性只能与 X 轴方向的锚点属性交互；
* Y 轴方向的锚点属性只能与 Y 轴方向的锚点属性交互；
* 尺寸锚点属性只能与尺寸锚点属性交互。

下面我们对比一下两种创建约束的方式：

1. 使用 `NSLayoutConstraint` 直接创建约束
2. 使用 `NSLayoutAnchor` 间接创建约束

```objc
// 1.Creating constraints using NSLayoutConstraint
NSLayoutConstraint(item: subview,
                   attribute: .leading,
                   relatedBy: .equal,
                   toItem: view,
                   attribute: .leadingMargin,
                   multiplier: 1.0,
                   constant: 0.0).isActive = true

NSLayoutConstraint(item: subview,
                   attribute: .trailing,
                   relatedBy: .equal,
                   toItem: view,
                   attribute: .trailingMargin,
                   multiplier: 1.0,
                   constant: 0.0).isActive = true

// 2. Creating the same constraints using Layout Anchors
let margins = view.layoutMarginsGuide

subview.leadingAnchor.constraint(equalTo: margins.leadingAnchor).isActive = true
subview.trailingAnchor.constraint(equalTo: margins.trailingAnchor).isActive = true
	
```

如下所示为 `NSLayoutAnchor` 提供的一些间接创建约束的方法。

```objc
// 继承自 NSLayoutAnchor 的共有 API
func constraint(equalTo anchor: NSLayoutAnchor<AnchorType>) -> NSLayoutConstraint
func constraint(equalTo anchor: NSLayoutAnchor<AnchorType>, constant c: CGFloat) -> NSLayoutConstraint

// NSLayoutXAxisAnchor 的 API
func constraint(equalToSystemSpacingAfter anchor: NSLayoutXAxisAnchor, multiplier: CGFloat) -> NSLayoutConstraint
func anchorWithOffset(to otherAnchor: NSLayoutXAxisAnchor) -> NSLayoutDimension

// NSLayoutYAxisAnchor 的 API
func constraint(equalToSystemSpacingBelow anchor: NSLayoutYAxisAnchor, multiplier: CGFloat) -> NSLayoutConstraint
func anchorWithOffset(to otherAnchor: NSLayoutYAxisAnchor) -> NSLayoutDimension

// NSLayoutDimension 的 API
func constraint(equalTo anchor: NSLayoutDimension, multiplier m: CGFloat) -> NSLayoutConstraint
func constraint(equalTo anchor: NSLayoutDimension, multiplier m: CGFloat, constant c: CGFloat) -> NSLayoutConstraint
func constraint(equalToConstant c: CGFloat) -> NSLayoutConstraint
```

## VFL

![[VFL#简介]]

[[VFL#语法]]

[[VFL#创建约束]]


下面，我们来看一个使用 VFL 创建约束的例子。

```objc

NSNumber *left = @50;
NSNumber *top = @50;
NSNumber *width = @100;
NSNumber *height = @100;
NSDictionary *views = NSDictionaryOfVariableBindings(view1, view2);
NSDictionary *metrics = NSDictionaryOfVariableBindings(left, top, width, height);
[view1 addConstraints:[NSLayoutConstraint constraintsWithVisualFormat:@"H:|-left-[view(>=width)]" options:0 metrics:metrics views:views]];

```

# 布局因素

布局的构建主要由 **布局引擎** （Layout Engine）完成。毫无疑问，视图是构建布局的作用对象。约束作为自动布局的核心，是构建布局的重要依据。除此之外，布局引擎在构建布局时还会参考以下这些因素：

* 约束优先级（Constraint Priorities）
* **内容优先级** （Content Priorities）
* **固有内容尺寸** （Intrinsic Content Size）
* **尺寸约束** （Sizing Constraints）
* 水平对齐（Horizontal Alignment）
* 垂直对齐（Vertical Alignment）
* 基线对齐（Baseline Alignment）
* 对齐矩形（Alignment Rect）

![[16ea093508925870.jpeg]]

## 尺寸约束

事实上，在上文 **约束创建** 中创建的约束就已经包含了尺寸约束。这里的再次提到尺寸约束，主要是针对 Self-Sizing 的视图。

比如，我们可以通过自动布局自动计算 TableView 的 Cell 高度。不过，默认情况下未启用该功能。

默认情况下，TabelView 的 Cell 高度由协议声明的 `tableView:heightForRowAtIndexPath:` 方法确定。除此之外，我们可以通过对 TabeView 的两个属性赋值，从而启用 Self-Sizing 功能，如下所示：

```objc
tableView.estimatedRowHeight = 85.0
tableView.rowHeight = UITableViewAutomaticDimension
```

接下来，我们需要在 TableView 的 Cell 的 `contentView` 中进行布局。为了能让布局引擎自动计算出 Cell 的高度，我们必须对 `contentView` 的子视图在垂直方向上定义一系列完善的约束，尤其是高度约束。在布局引擎计算高度过程中，它会优先使用尺寸约束，其次它会使用固有内容尺寸。

## 固有内容尺寸 & 内容优先级

iOS 中有部分视图具有固有内容尺寸（intrinsic content size），固有内容尺寸就是视图内容和边距所占据的尺寸。比如， `UIButton` 的固有内容尺寸等于 Title 的尺寸加上内容边距（margin）。

具有固有内容尺寸的视图有以下这些：

View Intrinsic Content Size  Sliders Defines only the width (iOS).Defines the width, the height, or both—depending on the slider’s type (OS X).   Labels, buttons, switches, and text fields Defines both the height and the width.   Text views and image views Intrinsic content size can vary.

固有内容尺寸的大小受很多因素的影响。以 `UITextView` 为例，其固有内容尺寸的大小取决内容、是否启用了滚动、以及应用于 `UITextView` 的其他约束。如果可以滚动，则没有固有内容尺寸，如果不可滚动，则取决于所有文字的尺寸。

固有内容尺寸的大小还受内容优先级的影响，内容优先级有以下两个方面：

* **`Content Hugging Priority`**
* **`Content Compression Resistance Priority`**

`Content Hugging Priority` ：表示一个视图抗拉伸的优先级，数值越高优先级越高，越不容易被拉伸。

`Content Compressing Priority` ：表示一个视图抗压缩的优先级，数值越高优先级越高，越不容易被压缩。

![[16ea0935359fa332.jpeg]]

默认情况下，视图的 `Content Hugging Priority` 值是 `250` ， `Content Compression Resistance Priority` 值是 `750` 。因此，拉伸视图比压缩视图更容易。

### Intrinsic Content Size 与 Fitting Size 的关系

Intrinsic Content Size 是布局引擎的输入，基于此可以生成约束，并最终生成布局； Fitting Size 则相反，它是布局引擎的输出，是基于约束生成的布局结果。

## 对齐方式

对齐方式有三种类型：

* 水平对齐
* 垂直对齐
* 基线对齐

对于前两者，通过前文的描述我们也算是有所了解了。水平对齐，用于在 X 轴上产生约束；垂直对齐，用于在 Y 轴上产生约束。

基线对齐则是文本专有的一种专有的对齐方式。基线对齐包括 `firstBaseline` 和 `lastBaseline` 两种对齐方式。如下所示：

![[16ea093536bccfd0.jpeg]]

## 对齐矩形

在自动布局中，我们可能会认为约束是使用 `frame` 来确定视图的大小和位置的，但实际上，它使用的是 **对齐矩形** （alignment rect）。在大多数情况下， `frame` 和 `alignment rect` 是相等的，所以我们这么理解也没什么不对。

那么为什么是使用 `alignment rect` ，而不是 `frame` 呢？

有时候，我们在创建复杂视图时，可能会添加各种装饰元素，如：阴影，角标等。为了降低开发成本，我们会直接使用设计师给的切图。如下所示：

![[16ea093535d15b70.jpeg]]

其中，(a) 是设计师给的切图，(c) 是这个图的 `frame` 。显然，我们在布局时，不想将阴影和角标考虑进入（视图的 `center` 和底边、右边都发生了偏移），而只考虑中间的核心部分，如图 (b) 中框出的矩形所示。

对齐矩形就是用来处理这种情况的。 `UIView` 提供了方法可以实现从 `frame` 得到 `alignment rect` 以及从 `alignment rect` 得到 `frame` 。

```objc
// The alignment rectangle for the specified frame.
- (CGRect)alignmentRectForFrame:(CGRect)frame;

// The frame for the specified alignment rectangle.
- (CGRect)frameForAlignmentRect:(CGRect)alignmentRect;

```

此外，系统还提供了一个简便方法，有 `UIEdgeInsets` 指定 `frame` 和 `alignment rect` 的关系。

```objc
// The insets from the view’s frame that define its alignment rectangle.
- (UIEdgeInsets)alignmentRectInsets;

```

如果希望 `alignment rect` 比 `frame` 的下边多 `10` 个点，可以这些写：

```objc
- (UIEdgeInsets)alignmentRectInsets {
    return UIEdgeInsetsMake(.0, .0, -10.0, .0);
}

```

# 布局渲染

iOS 的布局渲染可以分为三个阶段，如下所示：

1. **约束更新** （Constraints Update）
2. **布局更新** （Layout Update）
3. **显示重绘** （Display Redraw）

![[16ea09353b06216e.jpeg]]

其中，每一步都是依赖前一步操作。显示重绘依赖布局更新，布局更新依赖约束更新。

## 约束更新

约束更新是 **自下而上** （从子视图到父视图）进行的。我们可以通过调用 `setNeedsUpdateConstraints` 来触发约束更新。当然，我们对布局因素（约束/内容优先级、约束、固有内容尺寸...）作出的任何修改都会 **自动触发** `setNeedsUpdateConstraints` 方法。

对于自定义视图，我们可以在约束更新阶段重写 `updateConstraints` 来为视图增加需要的本地约束。

## 布局更新

布局更新是 **自上而下** （从父视图到子视图）进行的。事实上，布局更新操作是通过设置 `frame` （OS X ）或 `center` 和 `bounds` （iOS）将布局引擎的计算结果应用到视图上。我们可以通过条用 `setNeedsLayout` 来触发布局更新。这并不会立刻应用布局，而是延迟进行处理。因为所有的布局请求将会被合并到一个布局操作中。这种延迟处理的过程被称为 `Deferred Layout Pass` 。

我们可以调用 `layoutIfNeeded` （iOS） 或 `layoutSubtreeIfNeeded` （OS X）强制系统立即更新视图树的布局。如果我们下一步的操作依赖于更新后视图的 `frame` ，这将非常有用。

对于自定义视图，我们可以布局更新阶段重写 `layoutSubviews` （iOS）或 `layout` （OS X）来获取控制布局变化的所有权。

## 显示重绘

显示重绘时 **自上而下** （从父视图到子视图）进行的。我们可以通过调用 `setNeedsDisplay` 来触发显示重绘，这回导致所有的调用都被合并到一起延迟重绘。

对于自定义视图，我们可以在显示重绘阶段重新 `drawRect:` 来获取自显示过程的所有权。

## 注意事项

要注意的是，这三个阶段并不是单向的。基于约束的布局是一个迭代的过程。布局更新可以基于之前的布局来对约束作出修改，而这将再次触发约束更新，并紧接另一个布局更新。这可以被用来创建高级的自定义视图布局。但是如果我们每一次调用的自定义 `layoutSubviws` 都会导致另一个布局操作的话，将会陷入无限循环中。

# 不同版本 iOS 的自动布局

## iOS 6

* 苹果在这个版本引入了自动布局，具有了所有核心功能。

## iOS 7

* NavigationBar、TabBar、ToolBar 的 `translucent` 属性默认为 `YES` 。当前 ViewController 的高度是整个屏幕的高度，为了确保不被这些 Bar 覆盖，可以在布局中使用 `topLayoutGuide` 和 `bottomLayoutGuide` 属性。

## iOS 8

* Self-Sizing Cells。传送门: [www.appcoda.com/self-sizing…](https://www.appcoda.com/self-sizing-cells/)
* `UIViewController` 新增两个方法，用来处理 `UITraitEnvironment` 协议。UIKit 里有 `UIScreen`、`UIViewController`、`UIPresentationController` 支持该协议。当视图 traitCollection 改变时， `UIViewController` 可以捕获到这个消息进行处理。

```objc
- (void)setOverrideTraitCollection:(UITraitCollection *)collection forChildViewController:(UIViewController *)childViewController NS_AVAILABLE_IOS(8_0);
- (UITraitCollection *)overrideTraitCollectionForChildViewController:(UIViewController *)childViewController NS_AVAILABLE_IOS(8_0);
```

* Size Class。 `UIViewController` 提供了一组新协议支持 `UIContentContainer` 。

```objc
- (void)systemLayoutFittingSizeDidChangeForChildContentContainer:(id )container NS_AVAILABLE_IOS(8_0);
- (CGSize)sizeForChildContentContainer:(id )container withParentContainerSize:(CGSize)parentSize NS_AVAILABLE_IOS(8_0);
- (void)viewWillTransitionToSize:(CGSize)size withTransitionCoordinator:(id )coordinator NS_AVAILABLE_IOS(8_0);
- (void)willTransitionToTraitCollection:(UITraitCollection *)newCollection withTransitionCoordinator:(id )coordinator NS_AVAILABLE_IOS(8_0);
```

* `UIView` 的 margin 新增了 3 个 API， `NSLayoutMargins` 可以定义视图之间的距离。只对自动布局有效，并且默认值为 `{8, 8, 8, 8}` 。 `NSLayoutAttribute` 的枚举值也有相应的更新。

```objc
// UIView的3个Margin相关API
@property (nonatomic) UIEdgeInsets layoutMargins NS_AVAILABLE_IOS(8_0);
@property (nonatomic) BOOL preservesSuperviewLayoutMargins NS_AVAILABLE_IOS(8_0);
- (void)layoutMarginsDidChange NS_AVAILABLE_IOS(8_0);

// NSLayoutAttribute的枚举值更新
NSLayoutAttributeLeftMargin NS_ENUM_AVAILABLE_IOS(8_0),
NSLayoutAttributeRightMargin NS_ENUM_AVAILABLE_IOS(8_0),
NSLayoutAttributeTopMargin NS_ENUM_AVAILABLE_IOS(8_0),
NSLayoutAttributeBottomMargin NS_ENUM_AVAILABLE_IOS(8_0),
NSLayoutAttributeLeadingMargin NS_ENUM_AVAILABLE_IOS(8_0),
NSLayoutAttributeTrailingMargin NS_ENUM_AVAILABLE_IOS(8_0),
NSLayoutAttributeCenterXWithinMargins NS_ENUM_AVAILABLE_IOS(8_0),
NSLayoutAttributeCenterYWithinMargins NS_ENUM_AVAILABLE_IOS(8_0),

```

## iOS 9

* `UIStackView` ：提供了更加简单的自动布局方法，如：alignment 的 fill，leading，center，trailing。distribution 的 fill，fill equally，fill proportionally，equal spacing。


* `NSLayoutAnchor` API

```objc
NSLayoutConstraint *constraint = [view1.leadingAnchor constraintEqualToAnchor:view2.topAnchor];
```

# 参考

1. [Solving Linear Arithmetic Constraints for User Interface Applications](https://constraints.cs.washington.edu/solvers/uist97.html)
2. [Visual Format Language](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/VisualFormatLanguage.html)
3. [Understanding Auto Layout](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/index.html)
4. [NSLayoutAnchor基础知识](https://peteruncle.com/2018/01/28/NSLayoutAnchor%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/)
5. [深入剖析Auto Layout，分析iOS各版本新增特性](https://ming1016.github.io/2015/11/03/deeply-analyse-autolayout/)
6. [iOS 自动布局基础知识](https://peteruncle.com/2018/01/08/iOS%E8%87%AA%E5%8A%A8%E5%B8%83%E5%B1%80%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/)
7. [WWDC 2015 session 218 Mysteries of Auto Layout Part1](https://developer.apple.com/videos/play/wwdc2015/218/)
8. [WWDC 2015 session 219 Mysteries of Auto Layout Part2](https://developer.apple.com/videos/play/wwdc2015/219/)
9. [WWDC 2018 session 220 High Performance Auto Layout](https://developer.apple.com/videos/play/wwdc2018/220)
10. [Advanced Auto Layout Toolbox](https://www.objc.io/issues/3-views/advanced-auto-layout-toolbox/)
11. [WWDC 2015 - 揭开AutoLayout的神秘面纱(Mysteries Of Auto Layout)](http://www.pluto-y.com/wwdc-2015-mystries-of-auto-layout/)

# 扩展阅读

1. [iOS 利用 AutoLayout 实现 view 间隔自动调整](https://juejin.im/entry/578354df1532bc005f5c1266)
2. [ios auto layout demystified (一)](https://www.cnblogs.com/lisa090818/p/4303426.html)
3. [先进的自动布局工具箱](https://objccn.io/issue-3-5/)
4. [关于UIView的translatesAutoresizingMaskIntoConstraints属性](https://samingzhong.github.io/2016/08/30/UIKit/%E5%85%B3%E4%BA%8EUIView%E7%9A%84translatesAutoresizingMaskIntoConstraints%20%E5%B1%9E%E6%80%A7/)
5. [Masonry](https://github.com/ming1016/study/wiki/Masonry)
6. [iOS 布局渲染——UIView 方法调用时机](https://segmentfault.com/a/1190000015300896)
7. [UIStackView基础知识](https://peteruncle.com/2017/08/09/UIStackView%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/)

(完)
