非常扯的一道面试题,真这么写代码会被人打死的

![[img_mianshiti.png]]

先说结果: 可以执行成功,打印结果是 viewController


再来分析为什么说和[[Runtime]]、[[OC 对象的本质]]、栈知识相关

先说 `void* obj = &cls` 这行代码,这里用到了OC对象本质的知识,搭配以下这张图来看
1. cls 储存了类对象的地址
2. *obj 储存了 cls 的地址值,cls 储存了MJPerson的地址值 isa也是存储了类的地址值
3. 所以这里 cls 起到了和instance 对象中的 isa 指针一样的作用
4. 这里的结论结束 *obj 是一个指向对象的指针,这个对象的 isa  指针是 cls, isa(cls) 指针指向了一个类
![[img_interview1.png]]

再来看 [(__bridge id)obj  print] 这行代码的执行,这里除了OC对象本质,还用到了runtime的知识
1. [obj  print] 在runtime 中会转换成通过isa指针(cls)找到类对象,然后在类对象方法列表中找到方法去执行.
2. 上面已经说了 cls 存储了类的值,通过 cls 是可以找到一个类对象的,所以 cls 在这里可以堪称对象中的isa 指针
3. 所以可以调用成功

至于为什么会打印 viewcontroller 这里用到了栈在内存中存储的相关知识,要看下面这张图
1. 在[[OC 对象的本质#如果一个实例对象中有属性的话 属性的值会包含在实例对象里]] 中我们知道, instance 对象的实质是一个结构体, 结构体中 isa 和对象的成员变量依次排列,这里 Person 的结构体大概是{isa,name}
2. 但是我们这个实例对象是假的, 它只有一个 isa, 但是 runtime 不知道,他仍然会通过 isa 的地址值 + 8 去寻找 _name 这个成员变量, 那会找到什么呢?
3. 需要注意的是这个欺骗性的对象不是和往常一样在堆中,而是在 viewdidload 这个方法的栈中,因为 cls 作为局部变量声明在这里
4. 因为栈中变量在内存中是自上而下添加,所以寻找 _name 的时候会找到在 cls 之前声明的变量
5. 在 cls 之前只声明了 viewController 对象, 因为 viewDidload 转换成 runtime 调用是 msgSend(id,SEL), 传入的id 是self,即 viewController
6. 所以会打印出 viewcontroller 
7. 如果在 cls 之前声明任何`对象`都会打印出这个变量
![[img_interview2.png]]








说完了,明白了,这个题涉及到的知识还是很多方面的,但又全是基础,叫人没脾气.
