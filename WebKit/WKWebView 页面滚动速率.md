WKWebView 需要通过scrollView delegate调整滚动速率：

```objc
- (void)scrollViewWillBeginDragging:(UIScrollView *)scrollView {
	scrollView.decelerationRate = UIScrollViewDecelerationRateNormal;
}
```