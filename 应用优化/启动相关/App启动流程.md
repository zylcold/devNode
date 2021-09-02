![[186f338fadad480ab914effa5971d13b~tplv-k3u1fbpfcp-zoom-1.image.png]]

# 启动 Pipeline
详细回顾下整个启动过程，以及各个阶段耗时的影响因素：

1. [[Before dyld阶段]]
	
		1. 点击图标，创建进程
		2. mmap 主二进制，找到 dyld 的路径
		3. mmap dyld，把入口地址设为 `_dyld_start`

2. [[dyld3 阶段]]

		1. 重启手机/更新/下载 App 的第一次启动，会创建启动闭包	
		2. 把没有加载的动态库 mmap 进来，动态库的数量会影响这个阶段	
		3. 对每个二进制做 bind 和 rebase，主要耗时在 Page In，影响 Page In 数量的是 objc 的元数据
		4. 初始化 objc 的 runtime，由于闭包已经初始化了大部分，这里只会注册 sel 和装载 category	
		5. +load 和静态初始化被调用，除了方法本身耗时，这里还会引起大量 Page In

3. [[UIKit init阶段]]

		1. 初始化 `UIApplication` ，启动 Main Runloop

4. [[AppLifeCycle阶段]]

       1. 执行 `will/didFinishLaunch` ，这里主要是业务代码耗时


5. [[First Frame Render阶段]]

       1. Layout， `viewDidLoad` 和 `Layoutsubviews` 会在这里调用， `Autolayout` 太多会影响这部分时间
		2. Display， `drawRect` 会调用
		3. Prepare，图片解码发生在这一步
		4. Commit，首帧渲染数据打包发给 RenderServer，启动结束

## [[dyld3启动速度提升的原因]]
