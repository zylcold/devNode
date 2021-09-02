#textkit #uikit 

> WWDC 2018 Session 221: [TextKit Best Practices](https://link.juejin.im?target=https%3A%2F%2Fdeveloper.apple.com%2Fvideos%2Fplay%2Fwwdc2018%2F221%2F)

> 作者简介： [@halohily](https://link.juejin.im?target=https%3A%2F%2Fweibo.com%2Fhalohily) ，网易有道 iOS 开发工程师。掘金主页： [halohily](https://juejin.im/user/5923121f8d6d810058ecdb21)

## 引言

文本内容在 app 内随处可见，展示文本的方式也是多种多样。关注过性能提升的同学会发现，文本控件的高效使用对于整个页面性能的提升至关重要。为此，苹果和开发者都在不断努力。比如苹果日渐完善的文本框架，以及第三方文本框架的代表 [YYText](https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fibireme%2FYYText) 。

这个 session 旨在指导开发者如何正确地使用 `TextKit` 进行文本内容的展示，循序渐进分为三个部分：

* 核心理论
* 用以演示理论的小例子
* 综合运用的优秀实战案例
## 一、核心理论

### 1.1 什么是 TextKit ？

和平时使用的框架有些不同，我们不需要使用 `import` 关键字来导入 `TextKit` 。包含 `UILabel`、`UITextField` 等控件的 `UIKit` 框架（用于 iOS），以及包含 `NSTextView` 等控件的 `AppKit` 框架（用于 Mac OS），都是基于 `TextKit` 构建。在使用上面的文本控件时，其实就是在使用 `TextKit` ，它协同 `Core Text`、`Core Graphics` 以及 `Foundation` ，一起为我们的 app 提供强大的文本展示能力。

![[1641167925b12ad2.jpeg]]

利用 `TextKit` 的能力，你可以非常容易地展示下面风格各异的文本。

![[164116812b2ac3a4.jpeg]]

### 1.2 选择正确的控件

对于不同类型的文本，我们需要选择合适的文本控件。那么该如何决定呢？苹果为我们提供了比较明确的指导，如下图所示。在使用 `UIKit` 和 `AppKit` 时，情形会稍有不同，所以分开进行描述。

* `UIKit` 的选择路径：

![[1641168f9e39cdbd.jpeg]]

* `AppKit` 的选择路径：

![[1641169724e97e73.jpeg]]

图中的描述非常清晰易读。需要注意的是， `UILabel` 用来展示较少的文本内容或者较少的行数，然而，在 `AppKit` 框架下是没有 `Label` 控件的，这时可以选择 `NSTextField` 控件，通过禁用文本编辑属性，来获得和 `UILabel` 一样的特性。

#### 1.2.1 文本绘制（string drawing）的正确使用

有的时候，大家可能为了获得更优的性能（避免生成过多的视图对象实例），通过调用如下方法来使用文本绘制：

```swift
func draw(at: CGPoint)

func draw(in: CGRect)

func draw(with: CGRect,
options: NSStringDrawingOptions = [],
context: NSStringDrawingContext?)
```

然而，苹果并不推荐经常这样使用。如果你依然需要使用的话，苹果也贴心地给出了一些建议：

* 尽量用于数量较少的文本
* 限制调用 `draw` 方法的频率（尽量减少调用次数）
* 限制定制化属性的数量（尽量减少定制化属性）
为什么这种使用方式不被推荐呢？首先是因为 `UILabel`、`UITextView` 等控件提供了良好的缓存机制，所以在合适的时候选择这些控件，反而可以获得更好的性能（相较 string drawing 而言），特别是在使用自动布局的时候。

绘制 `attributed string` 时，如果过多地调用 `draw` 方法，会明显地降低性能。因为系统在每次绘制之前需要释放之前所有的 `attribute` 对象。因此，对于额外的 `attribute` ，请尽量在确定它们的视觉效果（例如字体、颜色）时才进行绘制。

**最后，苹果还是不忘强调，如果使用了 string drawing，就会失去下图所示的文本控件提供的所有特性。因此，请尽可能地使用文本控件。**

![[164116b2d959a392.jpeg]]

### 1.3 选择正确的定制要点

#### 1.3.1 `TextKit` 的架构组成

像 `Cocoa` 下的许多组件一样， `TextKit` 也是基于 “model - view - controller” 设计结构的。并且这三层又各自包含 storage、layout、和 display 模块：

![[164116c0c3fe8403 1.jpeg]]

* **`Storage`**
深入了解一下各个部分的组成，首先是与 `Model` 层通信的 `Storage` 模块，它包含的 `NSTextStorage` 持有字符串的数据和属性信息。值得注意的是，它是 `MutableAttributedString` 的子类，因此使用方式和我们熟知的 `AttributedString` 一致。而 `NSTextContainer` 则负责模型化文本布局的地理位置、区域信息。


![[164116cff04c40bc 1.jpeg]]

* **`Display`**
接下来是 `Display` 模块，它和 `View` 层通信。这个模块我们通常关注的是文本控件的正确选择问题。

* **`Layout`**
最后是 `Layout` 模块，它和 `Controller` 层进行通信。 `NSLayoutManager` 是这个模块唯一的组成部分。它的强大让苹果用“野兽”来形容。 **它是整个展示过程的“大脑”，控制自己的布局过程。**

![[164116dfb4175b49 1.jpeg]]

#### 1.3.2 布局过程

这是文本布局过程的概览图：

![[164116e59a1e57b9.jpeg]]

* **属性修正**
文本布局发生在 `TextStorage` 进行属性修正之后。对于这个过程中的工作，举个例子，确保这段文本所选择的字体支持显示文本中的所有字符，如果发现不支持的字符，则进行相应替换。比如上图中的 `Tempura (天麩羅) is a tasty Japanese food. 🍤` 这段文本，字体指定了 `Times New Roman` 。然而，这个字体是不支持日语字符和 emoji 字符的。因此，在属性修正过程中，日语字符被指定了支持日语的 `Hiragino Mincho ProN` 字体，而 emoji 字符则被指定了 `Apple Color Emoji` 字体。

* **`glyph` 和 `character`**
属性修正完成后，布局过程就开始了。这里对上述概念的含义做一些说明。

`character` 中文译为“字符”，字符是可以转换为二进制存储的通用数据，而 `glyph` 可以译为字符的视觉表示符号。同一个 `character` 呈现在屏幕上，可以表现为不同的字体、视觉风格。而这些各异的视觉风格，就是由 `glyph` 来负责呈现， `glyph` 的生成，就是为指定了视觉效果（如字体）的字符确定展示所需的 `glyph` 的过程。下图是一个示例：

![[164117131de029a3.jpeg]]

可以看到， `character` 和 `glyph` 的对应关系不总是一对一的。图中的字符串 “ffi” 由三个字符组成，但整个字符串可以由一个 `glyph` 表示。再看下图的例子，一个单独的字符 “n”，也可以由两个 `glyph` 来表示。

![[1641171e785f7a4c.jpeg]]

> 关于这部分概念，提供一篇参考资料： [iOS 排版概念](https://www.jianshu.com/p/8853174f574b)

再回到布局过程的图示中来， `glyph` 布局，就是 `NSLayoutManager` 在视图上摆放 `glyph` 的过程。

### 1.4 选择正确的配置

如下图是 一个完整 `TextKit` 组件的标准配置结构： `Text Container` 持有 `Text View` 的弱引用，而 `Text View` 通过根 `Text Storage` 持有整个布局树结构。

![[1641172648ebeb98 1.jpeg]]

如果有多个文本页面或者文本行需要布局，可以使用成对的 `Text Container` 和 `Text View` 组合，每一对组合对应一个页面或者一行。在这种情况下，我们可以 hook 同样的 container 和 text view 来共享布局信息。

![[1641172b8181af0a 1.jpeg]]

文本内容被添加之后，它铺满由第一个 text container 定义的区域。文本在 text view 上和 text container 成对展示。 当没有剩余空间时，新的 container 连同 text view 一起被添加，并且文本在第二个页面或者文本行进行展示。

多个 layout manager 允许你对同样的文本有多种不同的显示效果。这个文本在不同的视图上可以有彼此不同且独立的布局和分组，下图是这种模式下的结构示意和效果示意。方框内的文本内容相同，但展示效果是不同的。

![[1641173bb442bccc 1.jpeg]]

### 1.5 选择正确的定制实现方式

就像锤子在工具箱中的重要地位一样，我们在开发时也有一些地位等同于锤子的工具。

* `代理` 就像基本的锤子，大多数时候，它可以很好地完成工作。
* `通知` 也是一个有效的工具。
* 最后， `子类化` 同样是一把利器。它几乎可以作任何事。
对于这些方式的使用场景，在第二部分会运用具体例子进行阐述。

## 二、具体示例

文本组件在 app 中是无处不在的。在这部分，苹果使用了 iOS 的 `Apple News` 和 Mac OS 的 `TextEdit` 、`Our Journal` 三个 app 中的具体页面作为示例来对前面所述的核心理论进行讲解。

### 2.1 Apple News on iOS

这部分内容比较简单。主要用来示意 `Choosing the right control` 这条理论。里面主要的知识点如下：

* 对于一行颜色不一样的文本，可以使用两个 `UILabel` 进行展示，也可以借助 `NSAttributedString` 来实现。
* `UITextView` 是 `UIScrollView` 的子类，默认支持滑动，如果想让它与自动布局良好协作地话，需要禁用滑动。
### 2.2 TextEdit on macOS

这部分主要用来示意 `Choosing the right configuration` 这条理论。

![[1641175fe77846b0.jpeg]]

`TextEdit` 这个 app 支持富文本的展示、编辑，文本编辑部分的特性很像一个 textview，自然，它符合前面讲述的标准配置结构。值得注意的是，文本编辑部分支持分页展示，可以看到页面下滑时，textcontainer 被重新设置了尺寸，文本从第一页跳到了第二页。很自然，这是使用了多个 textcontainer 的 textview，但是依然由同一个 textstorge、layoutmanager 管理，他们允许文本自由地从一个 textcontainer 跳到另一个。下图即是它的配置结构图：

![[1641177235355318.jpeg]]

### 2.3 Our Journal App on macOS

这部分主要用来示意 `Choosing the right customization approach` 这条理论。

![[1641177e18be8231.jpeg]]

#### 2.3.1 文本计数功能

从图中可以看到，在界面底部添加了一个 TextField 来显示键入文本的数量。app 运行时，我们希望底部的文本计数随着键入的数量变化。为了实现这个效果，我们选择一个比较“轻巧”的工具 - 通知。通过接收 NSTextStorage 发出的通知，可以从 NSTextStorage 获得文本的数量。收到通知后，更新计数 TextField 中的数字。

#### 2.3.2 自动转化粗体字

当我们想强调一部分文字时，可以使用键盘快捷键或者菜单设置这部分字体为粗体。但是如果想支持例如 `markdown` 的标记语言，通过特定字符来指定特殊的格式，比如在文本前后加入一对双星号来使文本变化为粗体，该如何实现呢？在这个情景中，需要获取文本改变的时机和位置，通知机制并不便于提供足够的信息。所以这次使用“一记重锤” - 代理。遵守 `NSTextStorageDelegate` 协议，实现 `textStorage(_:didProcessEditing:range:changeInLength:)` 方法。在方法的实现中定义一个粗体字的 attribute ，添加给应该被粗体化的文本。这样一来，只要输入了一对双星号，就可以立马使文本变为粗体。

![[1641178da14ffb25.jpeg]]

#### 2.3.3 代码片段文本

粗体标记完美实现了。那么如何展示一个代码片段呢？像图中所示，完成键入最后一个点符号，就可以生成一个代码块文本，同时还会被标示为 `Swift` 代码。对于这样一个复杂的情形，我们需要两把工具：

* 子类化 `NSTextStorage`
子类继承 `NSTextStorage` ，实现四个强制实现的方法，特别是 `replaceCharacters(in:with:)` 方法。内部实现是将 `NSTextBlock` 赋值给 `ParagraphStyle` 然后把这个 `ParagraphStyle` 作为一个 `attribute` 添加到一个 `NSTextStorage` 中，注意对应的范围是代码块文本。

![[16411795bdcc60f6.jpeg]]

对于上面所述的 `NSTextBlock` ，需要了解的是 `NSTextBlock` 不会去定制化绘制它自己，所以我们需要一个它的子类去完成这件事： `CodeBlock` 类继承自 `NSTextBlock` ，在它的初始化方法中设置背景的衬垫，或者通过覆写 `drawBackground` 方法，使用 StringDrawing 去绘制 “Swift Code” 这个标题。

![[1641179bc9812b3c.jpeg]]

这样一来这个文本块看起来就像一个代码块了。再回到继承自 `TextStorage` 的 `CustomTextStorage` ，我们可以把 `TextBlocks` 属性赋值为刚刚添加的 `CodeBlock` 。

![[164117a027949530.jpeg]]

最后，我们需要让 textview 使用全新的 `CustomTextStorage` ，所以我们为 `LayoutManager` 替换 storage。

![[164117a39ace04cf.jpeg]]

#### 2.3.4 markdown效果预览视图

这样一来，基本完成了一个支持 markdown 格式的编辑器。除此之外，一般 markdown 编辑器还有一个很实用的功能 - 两个并排布局的视图，一个用来输入文本，一个预览效果，如图所示：

![[164117aaeb3b2167.jpeg]]

我们可以使用两个并排的 textview 来实现，只需要禁用用于预览的 textview 的文本编辑功能。它们展示一样的内容，但是右边的样式会特别一些。使用的配置如图：

![[164117ae88ed22fd.jpeg]]

storage 是同一个，因为展示一样的内容。但是其他的部分都是两套，并且用左边 view 的 textstorage 为右边 view 的 layoutmanager 的 `replaceTextStorage` 赋值。这样的效果是什么呢？一旦在一边编辑了文本，效果会在两边同时展示。但是一般在预览视图内我们是不希望显示 markdown 格式控制相关字符的，比如双星号 `**` 和 引用符号 `>` 等。由于是共享的同一个 textstorge，这就意味着我们必须在后面的过程中（布局过程）隐藏这些字符。为了完成这个操作，就有了一个自然而然的选项--代理：遵守 `NSLayoutManagerDelegate` 代理协议，实现 `layoutManager(_:shouldGenerateGlyphs:properties: characterIndexes:font:forGlyphRange:)` 代理方法，我们可以获取到将要被布局的 glyphs，如果它是用来表示 markdown 字符的 glyph，把它赋值为空。最后，把处理过的 glyphs 回传。这样一来，左边展示可编辑的包含 markdown 控制字符的文本，右边展示去除了 markdown 控制字符的效果文本。虽然事实上一个 markdown 编辑器并不是这样处理，但这是一个定制 `TextKit` 的很好的例子。

## 三、最佳实战案例

在这部分中，苹果给出了几个指导性原则。

### 3.1 熟知默认 attribute

![[164117baaf3d4c9a.jpeg]]


在这个例子中，我们需要完成一个如上图的文本展示。它当前的字体是 24 号的 `Comic Sans MS` 。给 `don't` 这部分文本设置粗体的 attribute 之后，我们发现剩余的文本（即 `hate` ）丢失了原本的字体设置。这是因为初始化 `AttributedString` 时，没有提供 attribute 设置参数，那么系统便会使用默认的设置。在这个案例中，使用默认设置初始化了文本，然后对 `don't` 部分进行了单独设置，自然 `hate` 部分就使用了默认的设置。

![[164117c1d2022bcc.jpeg]]

我们有两种方式来解决这件事。一种是避免将整个文本同时进行设置，而是对于 `don‘t` 设置粗体，对于 `hate` 设置 `Comic Sans MS` ，但这样比较繁琐。所以另一种是初始化 AttributedString 时，附带原有字体的参数，然后对 `don‘t` 部分再行设置。

![[164117c552f4a6b0.jpeg]]

除了字体外，我们还需要了解其他属性的默认值。

![[164117c952ba392c.jpeg]]

### 3.2 使用准确的属性描述

* 避免将全部或部分文本重置为默认属性的操作。
* 在更新你的 app 以支持即将到来的黑暗模式时，确保在这个模式下你的文本颜色正确。对于 appkit 开发者，这是非常重要的。
这里特别注意上图标记出的 `ParagraphStyle` 属性。一个反面案例是： 为了截掉 `hate` 部分的文本，给这部分文本单独设置了 `ParagraphStyle` 的属性。然而展示的结果却不符合预期。这是因为在 layout 之前，会进行 attribute fixing，这在前文有述。一个文本段落，却有多个 `ParagraphStyle` 的属性值，这是违反一致性的，所以系统在 fix attribute 时，会选择第一个 ParagraphStyle 属性，也就是默认风格，并且把它应用于整个段落。

### 3.3 性能表现：使用间断的布局

为了理解它，回到我们的老朋友 - 布局过程。glyph 生成之后进行 glyph 布局。对于大段文本，如果使用整体的布局，那么 LayoutManager 必须完成所有的 glyph 生成、布局过程，这样一来，如果有大段文本的话你就需要长时间地等待。

![[164117cd1d618e4d.jpeg]]

对于 `NSTextView` ，你可以通过设置 `allowsNonContiguousLayout` 属性来支持间断布局。

对于 `UITextView` ，它是默认开启的。需要注意的是， `UITextView` 是 `UIScrollView` 的子类， `allowsNonContiguousLayout` 属性要求 `UITextView` 的 `Scroll Enabled` 属性是开启的。因为如果不支持滑动的话，间断布局也就失去了意义。

这就引出了一个重要的问题。使用间断布局时，避免一次请求整个文本的布局。所以如果你只有一个 textcontainer 的话，避免一次请求完整的布局。

### 3.4 安全性

这里苹果给出了一个形象的例子：开发者就像武装的士兵，而 iOS、Mac OS 就像坚固的堡垒，士兵和堡垒共同组成了坚固的安全性防御工事。这就意味着，iOS 应用的安全性需要开发者和苹果共同协作。

![[164117d204a7b20b.jpeg]]

为此，苹果为开发者提供了一条准则：

* 为文本输入设置限制
所有的文本输入都被认为是潜在的风险。当你允许文本输入时，你就开放了复制和粘贴，但是你并不能预知什么文本会被粘贴在那里。它可能是一段普通的文本，但也有可能是极其长的文本，而这将会导致你的 app 出现不可预知的问题。

如何完成对文本的输入进行验证呢？在 `UIKit` 下，使用 `UITextFieldDelegate` ，在 `AppKit` 下通过 `NSFormatter` 。

**值得期待的是，苹果预告了关于安全性提升的内容即将到来** 。

## 总结

最后，用一张图来总结这个 session 的内容：

![[164117d593a9bfeb.jpeg]]

> *查看更多 WWDC 18 相关文章请前往* [老司机x知识小集xSwiftGG WWDC 18 专题目录](https://link.juejin.im?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2F6f95f7631d36)

