NSGenericException这个异常最容易出现在foreach操作中，在for in循环中如果修改所遍历的数组，无论你是add或remove，都会出错 "for in",它的内部遍历使用了类似 Iterator进行迭代遍历，一旦元素变动，之前的元素全部被失效，所以在foreach的循环当中，最好不要去进行元素的修改动作，若需要修改，循环改为for遍历，由于内部机制不同，不会产生修改后结果失效的问题。

NSInternalInconsistencyException
不一致导致出现的异常
比如NSDictionary当做NSMutableDictionary来使用，从他们内部的机理来说，就会产生一些错误

```objc
NSMutableDictionary *info = method return to NSDictionary type;
[info setObject:@“sxm" forKey:@"name"];
```

比如xib界面使用或者约束设置不当