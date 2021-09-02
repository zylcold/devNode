# 启动优化

os_signpost 是 iOS 12 推出的用于在 instruments 里标记时间段的 API，性能非常高，可以认为对启动无影响。结合最开始讲的分阶段监控，我们可以在 Instrument 把启动划分成多个阶段，和其他模板一起分析具体问题：

![[4a4b3c8941174428bcae89c9d05b76b0~tplv-k3u1fbpfcp-zoom-1.image.png]]

结合 swizzle，os_signpost 可以发挥出意想不到的效果，比如 hook 所有的 load 方法，来分析对应耗时，又比如 hook UIImage 对应方法，来统计启动路径上用到的图片加载耗时。