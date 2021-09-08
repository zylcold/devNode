## 简介
Visual Format Language 是苹果推出的为了简化 Auto Layout 编码的 DSL（Domain-Specific Language）。与iOS 6 随着Auto Layout 一起推出。


## 语法

说明 	|				示例 			
-- | --
标准间隔  | 			[button]-[textField]
宽度约束  | 			[button(>=50)]
父视图约束 |  	 \|-50-[purpleBox]-50-\|
垂直布局 |  			V:[topField]-10-[bottomField]
视图对齐 | 		[maroonView][blueView]
优先级  | 			 [button(100@20)]
宽度相等 | 			[button(\==button2)]
多个条件 |          [flexibleButton(>=70,<=100)]

注意，创建 VFL 语句描述时需要注意以下几点：

-   `H:` 和 `V:` 每次只能使用一个
-   视图变量名出现在方括号中，如： `[blueView]`
-   语句的顺序：从上到下，从左到右
-   视图间隔以数字常量出现，如： `-10-`
-   `|` 表示父视图


## 创建约束

`NSLayoutConstraint` 类提供了相关的 API 允许通过 VFL 语句创建约束。

```objc
+ (NSArray<NSLayoutConstraint *> *)constraintsWithVisualFormat:(NSString *)format options:(NSLayoutFormatOptions)opts metrics:(NSDictionary<NSString *,id> *)metrics (NSDictionary<NSString *,id> *)views;
```


其中， `format` 表示 VFL 语句； `options` 表示约束类型； `metrics` 表示 VFL 语句中用到的具体数值； `views` 表示 VFL 语句中用到的控件。