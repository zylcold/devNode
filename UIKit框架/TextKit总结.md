#textkit #uikit 


## 简单的绘制一段文字
在屏幕上绘制一段文字，iOS提供了三种方式，string drawing, CoreText, TextKit
### string drawing
```swift
class YLLabel_Draw: UIView {
    override func draw(_ rect: CGRect) {
        super.draw(rect)
		   //1. 设置图形上下文
        let context = UIGraphicsGetCurrentContext()
        context?.addRect(rect)
        UIColor.white.setFill()
        context?.fillPath()

		   //2. 绘制文字
		   str.draw(in: rect, withAttributes: nil)
    }
}
let label = YLLabel_Draw(frame: CGRect(x: 0, y: 0, width: 100, height: 100))
label
```
1. 获取当前图形的上下文
2. 调用NSString/NSAttributedString的draw方法。

NSString相关的绘制方法：
```swift
func draw(at: CGPoint, withAttributes: [NSAttributedString.Key : Any]?)
func draw(in: CGRect, withAttributes: [NSAttributedString.Key : Any]?)
func draw(with: CGRect, options: NSStringDrawingOptions, attributes: [NSAttributedString.Key : Any]?, context: NSStringDrawingContext?)
```

NSAttributedString相关绘制方法
```swift
func draw(at: CGPoint)
func draw(in: CGRect)
func draw(with: CGRect, options: NSStringDrawingOptions, context: NSStringDrawingContext?)
```

### CoreText
1. 需要翻转坐标系

```swift
class YLLabel_CoreText: UIView {
    override func draw(_ rect: CGRect) {
        super.draw(rect)
        guard let context = UIGraphicsGetCurrentContext() else { return }
        
		   //1. 设置背景
        context.addRect(rect)
        UIColor.white.setFill()
        context.fillPath()
        
        let text = str as CFString
        guard let attributedString = CFAttributedStringCreate(kCFAllocatorDefault, text, nil) else {
            debugPrint("create attributedString fail")
            return
        }
	      
        //2. 反转坐标系
        context.translateBy(x: 0, y: self.bounds.size.height)
        context.scaleBy(x: 1.0, y: -1.0)
        context.textMatrix = .identity
        
        let framesetter = CTFramesetterCreateWithAttributedString(attributedString)
		  //3. 创建路径
        let path = CGMutablePath()
        path.addRect(rect)
        let frame = CTFramesetterCreateFrame(framesetter, .init(location: 0, length: 0), path, nil)
        CTFrameDraw(frame, context)
    }
}
```
1. 获取当前图形的上下文
2. 反转坐标系
3. 创建framesetter
4. 创建绘制路径
5. 创建CTFrame
6. 绘制

### TextKit

```swift
class YLLabel_TextKit: UIView {
    override func draw(_ rect: CGRect) {
        super.draw(rect)
        guard let context = UIGraphicsGetCurrentContext() else { return }

		   //1. 设置背景       
        context.addRect(rect)
        UIColor.white.setFill()
        context.fillPath()
        
        let textContainer = NSTextContainer(size: rect.size)
        let attr = NSAttributedString(string: str as String, attributes: nil)
        let textStonge = NSTextStorage(attributedString: attr)
        let layoutManger = NSLayoutManager()
		  [layoutManager addTextContainer:textContainer];
        layoutManger.addTextContainer(textContainer)
        //2. 绘制
        layoutManger.drawGlyphs(forGlyphRange: NSRange(location: 0, length: str.length), at: CGPoint.zero)
    }
}
```
1. 获取当前图形的上下文
2. 创建NSTextContainer
3. 创建textStonge
4. 创建layoutManger
5. 关联textContainer，textStonge
6. 绘制

## TextKit
iOS7 之前的架构模式

[image:65698075-2D5E-45B3-90CF-54E7A7EF6E5B-1690-00005EC03174D1F3/TextRenderingArchitecture-iOS6.png]
![[TextRenderingArchitecture-iOS6.png]]

iOS7 之后的架构模式

[image:169A841E-1543-4483-A373-F4F817C0CF5B-1690-00005EC4BB7805F7/TextRenderingArchitecture-iOS7.png]
![[TextRenderingArchitecture-iOS7.png]]

默认支持的功能
* Dynamic type(动态字体)
 [image:4E8AA2E9-8DE1-4511-9D02-5641C66AFDAA-38525-0001C4D6E49E6F79/UserTextPreferences.png]
 ![[UserTextPreferences.png]]
 
* Letterpress effects(特殊效果-凸印效果)
[image:1AC18E10-A62D-40BB-8FDC-8940C0CE87FE-422-000051C010ECF8B7/4F027BE9-7A15-4CB1-9414-2B9854A6B563.png]
![[4F027BE9-7A15-4CB1-9414-2B9854A6B563.png]]

* Exclusion paths(排除路径显示)
[image:4B644330-CA22-46F6-89A8-4DD3EF4EE81A-38525-0001C4E8D38D51EB/BoldText.png]
![[BoldText.png]]
* Dynamic text formatting and storage(字体动态格式化显示和存储)


## TextKit的MVC
TextKit也是按照MVC[Model-View-Controller](https://developer.apple.com/library/archive/documentation/General/Conceptual/DevPedia-CocoaCore/MVC.html)的模式设计的
[image:189B5BB7-A80E-4570-94B2-E5F2128B5925-422-00003CD65D61E801/164116c0c3fe8403.jpeg]
![[164116c0c3fe8403 1.jpeg]]

### Model
`Storage` 模块：`NSTextStorage`、`NSTextContainer`

`NSTextStorage` 持有字符串的数据和属性信息。它是 ::`MutableAttributedString` 的子类::，因此使用方式和我们熟知的 `AttributedString` 一致。
`NSTextContainer` 则负责模型化文本布局的地理位置、区域信息。

[image:397C3EF5-9961-497C-A063-B0F2C7E22F71-422-00003CEB25205F0E/164116cff04c40bc.jpeg]
![[164116cff04c40bc 1.jpeg]]


### View
这个模块我们通常关注的是文本控件的正确选择问题。

### Controller
`NSLayoutManager` 是这个模块唯一的组成部分。 **它是整个展示过程的“大脑”，控制自己的布局过程。**
[image:B5CD5E31-43A0-4AFC-BA06-390C8C1C8FED-422-00003CFE883511EB/164116dfb4175b49.jpeg]
![[164116dfb4175b49 1.jpeg]]

### NSTextStorage、NSTextContainer、 NSLayoutManager关系
一个TextStorage可对应一/多个NSLayoutManager，NSLayoutManager也可以对应一/多个NSTextContainer

#### 1:1:1
标准配置结构： `Text Container` 持有 `Text View` 的弱引用，而 `Text View` 通过根 `Text Storage` 持有整个布局树结构。

[image:17340E47-509D-459E-ADEA-431286E3187D-422-00003D8AC6CF8793/1641172648ebeb98.jpeg]
![[1641172648ebeb98 1.jpeg]]

#### 1:1:N
同一文本需要跨页\不同区域显示

文本内容被添加之后，它铺满由第一个 text container 定义的区域。文本在 text view 上和 text container 成对展示。 当没有剩余空间时，新的 container 连同 text view 一起被添加，并且文本在第二个页面或者文本行进行展示。
[image:D9345363-D13C-4B49-85AB-745C36222991-422-00003D931C15C876/1641172b8181af0a.jpeg]
![[1641172b8181af0a 1 1.jpeg]]

#### 1:N:N 
文本相同，显示效果不同

多个 layout manager 允许你对同样的文本有多种不同的显示效果。这个文本在不同的视图上可以有彼此不同且独立的布局和分组
[image:6AB74811-300A-414B-844A-3266A5272D24-422-00003D9C29D52F76/1641173bb442bccc.jpeg]
![[1641173bb442bccc 1.jpeg]]

### NSTextContainer(文本容器)
主要负责回答NSLayoutManager(布局管理器)如下问题：对于给定的矩形，哪个部分可以放文字？
由下面的方法回答此问题
```objectivec
/**
@Parameters 
proposedRect
用于布局的矩形
characterIndex
正在处理片段在字符中的位置
baseWritingDirection
布局的水平方向. 通常是 NSWritingDirectionLeftToRight 或 NSWritingDirectionRightToLeft.
remainingRect
当前返回矩形内的排除矩形
*/
- (CGRect)lineFragmentRectForProposedRect:(CGRect)proposedRect atIndex:(NSUInteger)characterIndex writingDirection:(NSWritingDirection)baseWritingDirection remainingRect:(nullable CGRect *)remainingRect
```

#### 排除路径

```objectivec
UIBezierPath *path = [UIBezierPath bezierPathWithRect:CGRectMake(40, 40, 40, 40)];
textContainer.exclusionPaths = @[path];
```


### NSTextStorage(文本存储)
NSTextStorage是NSMutableAttributedString子类，它其实一个NSMutableAttributedString的一个“半具体”子类。部分NSMutableAttributedString原始方法并没有实现。以下方法是必须要实现的：
```objectivec
- (NSString *)string;
- (NSDictionary *)attributesAtIndex:(NSUInteger)location effectiveRange:(NSRangePointer)range;
- (void)replaceCharactersInRange:(NSRange)range withString:(NSString *)str;
- (void)setAttributes:(NSDictionary *)attrs range:(NSRange)range;
```
其中最重要的方法是::-(void) processEditing;::文本每次变更时，都会通知此方法。

#### 通知
```objectivec

typedef NS_OPTIONS(NSUInteger, NSTextStorageEditActions) {
    NSTextStorageEditedAttributes = (1 << 0), //属性变更
    NSTextStorageEditedCharacters = (1 << 1)  //字符变更
}

//文本变更
- (void)edited:(NSTextStorageEditActions)editedMask range:(NSRange)editedRange changeInLength:(NSInteger)delta;
```

能做：
为添加文本设置属性
不能做：
改变属性的绘制方式，变更文本内容

### NSLayoutManager(布局管理器)
布局管理器是中心组件，它把所有组件粘合在一起：
1、这个管理器监听文本存储中文本或属性改变的通知，一旦接收到通知就触发布局进程。
2、从文本存储提供的文本开始，它将所有的字符翻译为字形（Glyph）
3、一旦字形全部生成，这个管理器向它的文本容器（们）查询文本可用以绘制的区域
4、然后这些区域被行逐步填充，而行又被字形逐步填充。一旦一行填充完毕，下一行开始填充。
5、对于每一行，布局管理器必须考虑断行行为（放不下的单词必须移到下一行）、连字符、内联的图像附件等等。
6、当布局完成，文本的当前显示状态被设为无效，然后文本管理器将前面几步排版好的文本设给文本视图。

绘制文本
```objectivec
//绘制背景
- (void)drawBackgroundForGlyphRange:(NSRange)glyphsToShow atPoint:(CGPoint)origin;
//绘制字型
- (void)drawGlyphsForGlyphRange:(NSRange)glyphsToShow atPoint:(CGPoint)origin;
```




## 参考
[About Text Handling in iOS](https://developer.apple.com/library/archive/documentation/StringsTextFonts/Conceptual/TextAndWebiPhoneOS/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009542-CH1-SW1)
[iOS 7系列译文：认识 TextKit](bear://x-callback-url/open-note?id=4BFDB0BD-45FD-4E78-A97D-1C935AD27F50-1690-0000068729311F16)
[Text Kit Tutorial](bear://x-callback-url/open-note?id=DA468FE9-5DF8-4035-AF14-AD820EF5BCD5-1146-0000007B47C4E352)
[WWDC 2018：TextKit 最佳实践](bear://x-callback-url/open-note?id=14A5486F-254D-45B3-B592-91027E0D0B32-6393-0000046BACA9C006)
[如何实现文本动画](bear://x-callback-url/open-note?id=8E0F1083-BF68-4CB9-A760-F2EB928AEBD9-6430-00000474F1DA64E0)
[随处使用Cocoa文字系统](bear://x-callback-url/open-note?id=C6F2F269-D0AB-4BC6-9508-885735C65A00-41917-0001169322935E8C)