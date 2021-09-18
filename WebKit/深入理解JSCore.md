#webkit 

动态化作为移动客户端技术的一个重要分支，一直是业界积极探索的方向。目前业界流行的动态化方案，如Facebook的React Native，阿里巴巴的Weex都采用了前端系的DSL方案，而它们在iOS系统上能够顺利的运行，都离不开一个背后的功臣：JavaScriptCore（以下简称JSCore），它建立起了Objective-C（以下简称OC）和JavaScript（以下简称JS）两门语言之间沟通的桥梁。无论是这些流行的动态化方案，还是WebView Hybrid方案，亦或是之前广泛流行的JSPatch，JSCore都在其中发挥了举足轻重的作用。作为一名iOS开发工程师，了解JSCore已经逐渐成为了必备技能之一。

## 从浏览器谈起

在iOS 7之后，JSCore作为一个系统级Framework被苹果提供给开发者。JSCore作为苹果的浏览器引擎WebKit中重要组成部分，这个JS引擎已经存在多年。如果想去追本溯源，探究JSCore的奥秘，那么就应该从JS这门语言的诞生，以及它最重要的宿主-Safari浏览器开始谈起。

## JavaScript历史简介

JavaScript诞生于1995年，它的设计者是Netscape的Brendan Eich，而此时的Netscape正是浏览器市场的霸主。

而二十多年前，当时人们在浏览网页的体验极差，因为那会儿的浏览器几乎只有页面的展示能力，没有和用户的交互逻辑处理能力。所以即使一个必填输入框传空，也需要经过服务端验证，等到返回结果之后才给出响应，再加上当时的网速很慢，可能半分钟过去了，返回的结果是告诉你某个必填字段未填。所以Brendan **花了十天写出了JavaScript** ，由浏览器解释执行，从此之后浏览器也有了一些基本的交互处理能力，以及表单数据验证能力。

而Brendan可能没有想到，在二十多年后的今天。JS这门解释执行的动态脚本语言，不光成为前端届的“正统”，还入侵了后端开发领域，在编程语言排行榜上进入前三甲，仅次于Python和Java。而如何解释执行JS，则是各家引擎的核心技术。目前市面上比较常见的JS引擎有Google的V8（它被运用在Android操作系统以及Google的Chrome上），以及我们今天的主角–JSCore（它被运用在iOS操作系统以及Safari上）。

## WebKit

我们每天都会接触浏览器，使用浏览器进行工作、娱乐。让浏览器能够正常工作最核心的部分就是浏览器的内核，每个浏览器都有自己的内核，Safari的内核就是WebKit。WebKit诞生于1998年，并于2005年由Apple公司开源，Google的Blink也是在WebKit的分支上进行开发的。

WebKit由多个重要模块组成，通过下图我们可以对WebKit有个整体的了解：

![[c718ac49.png]]

简单点讲，WebKit就是一个页面渲染以及逻辑处理引擎，前端工程师把HTML、JavaScript、CSS这“三驾马车”作为输入，经过WebKit的处理，就输出成了我们能看到以及操作的Web页面。从上图我们可以看出来，WebKit由图中框住的四个部分组成。而其中最主要的就是WebCore和JSCore（或者是其它JS引擎），这两部分我们会分成两个小章节详细讲述。除此之外，WebKit Embedding API是负责浏览器UI与WebKit进行交互的部分，而WebKit Ports则是让Webkit更加方便的移植到各个操作系统、平台上，提供的一些调用Native Library的接口，比如在渲染层面，在iOS系统中，Safari是交给CoreGraphics处理，而在Android系统中，Webkit则是交给Skia。

### WebCore

在上面的WebKit组成图中，我们可以发现只有WebCore是红色的。这是因为时至今日,WebKit已经有很多的分支以及各大厂家也进行了很多优化改造，唯独WebCore这个部分是所有WebKit共享的。WebCore是WebKit中代码最多的部分，也是整个WebKit中最核心的渲染引擎。那首先我们来看看整个WebKit的渲染流程：

![[930a3bb4.png]]

首先浏览器通过URL定位到了一堆由HTML、CSS、JS组成的资源文件，通过加载器（这个加载器的实现也很复杂，在此不多赘述）把资源文件给WebCore。之后HTML Parser会把HTML解析成DOM树，CSS Parser会把CSS解析成CSSOM树。最后把这两棵树合并，生成最终需要的渲染树，再经过布局，与具体WebKit Ports的渲染接口，把渲染树渲染输出到屏幕上，成为了最终呈现在用户面前的Web页面。

## JSCore

### 概述

终于讲到我们这期的主角 – JSCore。JSCore是WebKit默认内嵌的JS引擎，之所以说是默认内嵌，是因为很多基于WebKit分支开发的浏览器引擎都开发了自家的JS引擎，其中最出名的就是Chrome的V8。这些JS引擎的使命都相同，那就是解释执行JS脚本。而从上面的渲染流程图我们可以看到，JS和DOM树之间存在着互相关联，这是因为浏览器中的JS脚本最主要的功能就是操作DOM树，并与之交互。同样的，我们也通过一张图看下它的工作流程:

![[3a0744df.png]]

可以看到，相比静态编译语言生成语法树之后，还需要进行链接，装载生成可执行文件等操作，解释型语言在流程上要简化很多。这张流程图右边画框的部分就是JSCore的组成部分：Lexer、Parser、LLInt以及JIT的部分（之所以JIT的部分是用橙色标注，是因为并不是所有的JSCore中都有JIT部分）。接下来我们就搭配整个工作流程介绍每一部分，它主要分为以下三个部分：词法分析、语法分析以及解释执行。

**PS：严格的讲，语言本身并不存在编译型或者是解释型，因为语言只是一些抽象的定义与约束，并不要求具体的实现，执行方式。这里讲JS是一门“解释型语言”只是JS一般是被JS引擎动态解释执行，而并不是语言本身的属性。**

### 词法分析 – Lexer

词法分析很好理解，就是把一段我们写的源代码分解成Token序列的过程，这一过程也叫分词。在JSCore，词法分析是由Lexer来完成（有的编译器或者解释器把分词叫做Scanner）。

这是一句很简单的C语言表达式：

```
sum = 3 + 2;
```

将其标记化之后可以得到下表的内容：

元素标记类型sum标识符=赋值操作符3数字+加法操作符2数字;语句结束

这就是词法分析之后的结果，但是词法分析并不会关注每个Token之间的关系，是否匹配，仅仅是把它们区分开来，等待语法分析来把这些Token“串起来”。词法分析函数一般是由语法分析器（Parser）来进行调用的。在JSCore中，词法分析器Lexer的代码主要集中在parser/Lexer.h、Lexer.cpp中。

#### 语法分析 – Parser

跟人类语言一样，我们讲话的时候其实是按照约定俗成，交流习惯按照一定的语法讲出一个又一个词语。那类比到计算机语言，计算机要理解一门计算机语言，也要理解一个语句的语法。例如以下一段JS语句：

```
var sum = 2 + 3;
var a = sum + 5;
```

Parser会把Lexer分析之后生成的token序列进行语法分析，并生成对应的一棵抽象语法树(AST)。这个树长什么样呢？在这里推荐一个网站： [esprima Parser](http://esprima.org/demo/parse.html) ，输入JS语句可以立马生成我们所需的AST。例如，以上语句就被生成这样的一棵树：

![[b2b87b5d.png]]

之后，ByteCodeGenerator会根据AST来生成JSCore的字节码，完成整个语法解析步骤。

### 解释执行 – LLInt和JIT

JS源代码经过了词法分析和语法分析这两个步骤，转成了字节码，其实就是经过任何一门程序语言必经的步骤–编译。但是不同于我们编译运行OC代码，JS编译结束之后，并不会生成存放在内存或者硬盘之中的目标代码或可执行文件。生成的指令字节码，会被立即被JSCore这台虚拟机进行逐行解释执行。

运行指令字节码（ByteCode）是JS引擎中很核心的部分，各家JS引擎的优化也主要集中于此。JSByteCode的解释执行是一套很复杂的系统，特别是加入了OSR和多级JIT技术之后，整个解释执行变的越来越高效，并且让整个ByteCode的执行在低延时之间和高吞吐之间有个很好的平衡：由低延时的LLInt来解释执行ByteCode，当遇到多次重复调用或者是递归，循环等条件会通过OSR切换成JIT进行解释执行（根据具体触发条件会进入不同的JIT进行动态解释）来加快速度。由于这部分内容较为复杂，而且不是本文重点，故只做简单介绍，不做深入的讨论。

### JSCore值得注意的Feature

除了以上部分，JSCore还有几个值得注意的Feature。

### 基于寄存器的指令集结构

JSCore采用的是基于寄存器的指令集结构，相比于基于栈的指令集结构（比如有些JVM的实现），因为不需要把操作结果频繁入栈出栈，所以这种架构的指令集执行效率更高。但是由于这样的架构也造成内存开销更大的问题，除此之外，还存在移植性弱的问题，因为虚拟机中的虚拟寄存器需要去匹配到真实机器中CPU的寄存器，可能会存在真实CPU寄存器不足的问题。

基于寄存器的指令集结构通常都是三地址或者二地址的指令集，例如：

```
i = a + b;
//转成三地址指令:
add i，a，b; //把a寄存器中的值和b寄存器中的值相加，存入i寄存器
```

在三地址的指令集中的运算过程是把a和b分别mov到两个寄存器，然后把这两个寄存器的值求和之后，存入第三个寄存器。这就是三地址指令运算过程。

而基于栈的一般都是零地址指令集，因为它的运算不依托于具体的寄存器，而是使用对操作数栈和具体运算符来完成整个运算。

### 单线程机制

值得注意的是，整个JS代码是执行在一条线程里的，它并不像我们使用的OC、Java等语言，在自己的执行环境里就能申请多条线程去处理一些耗时任务来防止阻塞主线程。JS代码本身并不存在多线程处理任务的能力。但是为什么JS也存在多线程异步呢？强大的事件驱动机制，是让JS也可以进行多线程处理的关键。

### 事件驱动机制

之前讲到，JS的诞生就是为了让浏览器也拥有一些交互，逻辑处理能力。而JS与浏览器之间的交互是通过事件来实现的，比如浏览器检测到发生了用户点击，会传递一个点击事件通知JS线程去处理这个事件。 那通过这一特性，我们可以让JS也进行异步编程，简单来讲就是遇到耗时任务时，JS可以把这个任务丢给一个由JS宿主提供的工作线程（WebWorker）去处理。等工作线程处理完之后，会发送一个message让JS线程知道这个任务已经被执行完了，并在JS线程上去执行相应的事件处理程序。（但是需要注意，由于工作线程和JS线程并不在一个运行环境，所以它们并不共享一个作用域，故工作线程也不能操作window和DOM。）

JS线程和工作线程，以及浏览器事件之间的通信机制叫做事件循环（EventLoop），类似于iOS的runloop。它有两个概念，一个是Call Stack，一个是Task Queue。当工作线程完成异步任务之后，会把消息推到Task Queue，消息就是注册时的回调函数。当Call Stack为空的时候，主线程会从Task Queue里取一条消息放入Call Stack来执行，JS主线程会一直重复这个动作直到消息队列为空。

![[f905fe0b.png]]
以上这张图大概描述了JSCore的事件驱动机制，整个JS程序其实就是这样跑起来的。这个其实跟空闲状态下的iOS Runloop有点像，当基于Port的Source事件唤醒runloop之后，会去处理当前队列里的所有source事件。JS的事件驱动，跟消息队列其实是“异曲同工”。也正因为工作线程和事件驱动机制的存在，才让JS有了多线程异步能力。

## iOS中的JSCore

> iOS7之后，苹果对WebKit中的JSCore进行了Objective-C的封装，并提供给所有的iOS开发者。JSCore框架给Swift、OC以及C语言编写的App提供了调用JS程序的能力。同时我们也可以使用JSCore往JS环境中去插入一些自定义对象。

iOS中可以使用JSCore的地方有多处，比如封装在UIWebView中的JSCore，封装在WKWebView中的JSCore，以及系统提供的JSCore。实际上，即使同为JSCore，它们之间也存在很多区别。因为随着JS这门语言的发展，JS的宿主越来越多，有各种各样的浏览器，甚至是常见于服务端的Node.js（基于V8运行）。随时使用场景的不同，以及WebKit团队自身不停的优化，JSCore逐渐分化出不同的版本。除了老版本的JSCore，还有2008年宣布的运行在Safari、WKWebView中的Nitro（SquirrelFish）等等。而在本文中，我们主要介绍iOS系统自带的JSCore Framework。

[iOS官方文档](https://developer.apple.com/documentation/javascriptcore?language=objc) 对JSCore的介绍很简单，其实主要就是给App提供了调用JS脚本的能力。我们首先通过JSCore Framework的15个开放头文件来“管中窥豹”，如下图所示：

![[a9a709c2.png]]

乍一看，概念很多。但是除去一些公共头文件以及一些很细节的概念，其实真正常用的并不多，笔者认为很有必要了解的概念只有4个：JSVM，JSContext，JSValue，JSExport。鉴于讲述这些概念的文章已经有很多，本文尽量从一些不同的角度（比如原理，延伸对比等）去解释这些概念。

### JSVirtualMachine

> 一个JSVirtualMachine（以下简称JSVM）实例代表了一个自包含的JS运行环境，或者是一系列JS运行所需的资源。该类有两个主要的使用用途：一是支持并发的JS调用，二是管理JS和Native之间桥对象的内存。

JSVM是我们要学习的第一个概念。官方介绍JSVM为JavaScript的执行提供底层资源，而从类名直译过来，一个JSVM就代表一个JS虚拟机，我们在上面也提到了虚拟机的概念，那我们先讨论一下什么是虚拟机。首先我们可以看看（可能是）最出名的虚拟机——JVM（Java虚拟机）。 JVM主要做两个事情：

1. 首先它要做的是把JavaC编译器生成的ByteCode（ByteCode其实就是JVM的虚拟机器指令）生成每台机器所需要的机器指令，让Java程序可执行（如下图）。
2. 第二步，JVM负责整个Java程序运行时所需要的内存空间管理、GC以及Java程序与Native（即C,C++）之间的接口等等。
![[89d86075.png]]

从功能上来看，一个高级语言虚拟机主要分为两部分，一个是解释器部分，用来运行高级语言编译生成的ByteCode，还有一部分则是Runtime运行时，用来负责运行时的内存空间开辟、管理等等。实际上，JSCore常常被认为是一个JS语言的优化虚拟机，它做着JVM类似的事情，只是相比静态编译的Java，它还多承担了把JS源代码编译成字节码的工作。

既然JSCore被认为是一个虚拟机，那JSVM又是什么？实际上，JSVM就是一个抽象的JS虚拟机，让开发者可以直接操作。在App中，我们可以运行多个JSVM来执行不同的任务。而且每一个JSContext（下节介绍）都从属于一个JSVM。但是需要注意的是每个JSVM都有自己独立的堆空间，GC也只能处理JSVM内部的对象（在下节会简单讲解JS的GC机制）。所以说，不同的JSVM之间是无法传递值的。

值得注意的还有，在上面的章节中，我们提到的JS单线程机制。这意味着，在一个JSVM中，只有一条线程可以跑JS代码，所以我们无法使用JSVM进行多线程处理JS任务。如果我们需要多线程处理JS任务的场景，就需要同时生成多个JSVM，从而达到多线程处理的目的。

### JS的GC机制

JS同样也不需要我们去手动管理内存。JS的内存管理使用的是GC机制（Tracing Garbage Collection）。不同于OC的引用计数，Tracing Garbage Collection是由GCRoot（Context）开始维护的一条引用链，一旦引用链无法触达某对象节点，这个对象就会被回收掉。如下图所示：

![[bf359c70.png]]

### JSContext

> 一个JSContext表示了一次JS的执行环境。我们可以通过创建一个JSContext去调用JS脚本，访问一些JS定义的值和函数，同时也提供了让JS访问Native对象，方法的接口。

JSContext是我们在实际使用JSCore时，经常用到的概念之一。”Context”这个概念我们都或多或少的在其它开发场景中见过，它最常被翻译成“上下文”。那什么是上下文？比如在一篇文章中，我们看到一句话：“他飞快的跑了出去。”但是如果我们不看上下文的话，我们并不知道这句话究竟是什么意思：谁跑了出去？他是谁？他为什么要跑？

写计算机理解的程序语言跟写文章是相似的，我们运行任何一段语句都需要有这样一个“上下文”的存在。比如之前外部变量的引入、全局变量、函数的定义、已经分配的资源等等。有了这些信息，我们才能准确的执行每一句代码。

同理，JSContext就是JS语言的执行环境，所有JS代码的执行必须在一个JSContext之中，在WebView中也是一样，我们可以通过KVC的方式获取当时WebView的JSContext。通过JSContext运行一段JS代码十分简单，如下面这个例子：

```objc
JSContext *context = [[JSContext alloc] init];
    [context evaluateScript:@"var a = 1;var b = 2;"];
    NSInteger sum = [[context evaluateScript:@"a + b"] toInt32];//sum=3
```

借助evaluateScript API，我们就可以在OC中搭配JSContext执行JS代码。它的返回值是JS中最后生成的一个值，用属于当前JSContext中的JSValue（下一节会有介绍）包裹返回。

我们还可以通过KVC的方式，给JSContext塞进去很多全局对象或者全局函数：

```objc
JSContext *context = [[JSContext alloc] init];
		context[@"globalFunc"] =  ^() {
        NSArray *args = [JSContext currentArguments];
        for (id obj in args) {
            NSLog(@"拿到了参数:%@", obj);
        }
    };
	context[@"globalProp"] = @"全局变量字符串";
   [context evaluateScript:@"globalFunc(globalProp)"];//console输出：“拿到了参数:全局变量字符串”
```

这是一个很好用而且很重要的特性，有很多著名的借助JSCore的框架如JSPatch，都利用了这个特性去实现一些很巧妙的事情。在这里我们不过多探讨可以利用它做什么，而是去研究它究竟是怎样运作的。在JSContext的API中，有一个值得注意的只读属性 – JSValue类型的globalObject。它返回当前执行JSContext的全局对象，例如在WebKit中，JSContext就会返回当前的Window对象。而这个全局对象其实也是JSContext最核心的东西，当我们通过KVC方式与JSContext进去取值赋值的时候， **实际上都是在跟这个全局对象做交互** ，几乎所有的东西都在全局对象里，可以说，JSContext只是globalObject的一层壳。对于上述两个例子，本文取了context的globalObject，并转成了OC对象，如下图：

![[6fb4ae62.png]]

可以看到这个globalObject保存了所有的变量与函数，这更加印证了上文的说法（至于为什么globalObject对应OC对象是NSDictionary类型，我们将在下节中讲述）。所以我们还能得出另外一个结论，JS中所谓的全局变量，全局函数不过是全局对象的属性和函数。

同时值得注意的是，每个JSContext都从属于一个JSVM。我们可以通过JSContext的只读属性 – virtualMachine获得当前JSContext绑定的JSVM。JSContext和JSVM是多对一的关系，一个JSContext只能绑定一个JSVM，但是一个JSVM可以同时持有多个JSContext。而上文中我们提到，每个JSVM同时只有整个一个线程来执行JS代码，所以综合来看，一次简单的通过JSCore运行JS代码，并在Native层获取返回值的过程大致如下：

![[cc094b9b.png]]

### JSValue

> JSValue实例是一个指向JS值的引用指针。我们可以使用JSValue类，在OC和JS的基础数据类型之间相互转换。同时我们也可以使用这个类，去创建包装了Native自定义类的JS对象，或者是那些由Native方法或者Block提供实现JS方法的JS对象。

在JSContext一节中，我们接触了大量的JSValue类型的变量。在JSContext一节中我们了解到，我们可以很简单的通过KVC操作JS全局对象，也可以直接获得JS代码执行结果的返回值（同时每一个JS中的值都存在于一个执行环境之中，也就是说每个JSValue都存在于一个JSContext之中，这也就是JSValue的作用域），都是因为JSCore帮我们用JSValue在底层自动做了OC和JS的类型转换。

JSCore一共提供了如下10种类型互换：

```
Objective-C type  |   JavaScript type
 --------------------+---------------------
         nil         |     undefined
        NSNull       |        null
       NSString      |       string
       NSNumber      |   number, boolean
     NSDictionary    |   Object object
       NSArray       |    Array object
        NSDate       |     Date object
       NSBlock 		   |   Function object 
          id         |   Wrapper object 
        Class        | Constructor object
```

同时还提供了对应的互换API（节选）：

```objc
+ (JSValue *)valueWithDouble:(double)value inContext:(JSContext *)context;
+ (JSValue *)valueWithInt32:(int32_t)value inContext:(JSContext *)context;
- (NSArray *)toArray;
- (NSDictionary *)toDictionary;
```

在讲类型转换前，我们先了解一下JS这门语言的变量类型。根据ECMAScript（可以理解为JS的标准）的定义：JS中存在两种数据类型的值，一种是基本类型值，它指的是简单的数据段。第二种是引用类型值，指那些可能由多个值构成的对象。基本类型值包括”undefined”，”nul”，”Boolean”，”Number”，”String”（是的，String也是基础类型），除此之外都是引用类型。对于前五种基础类型的互换，应该没有太多要讲的。接下来会重点讲讲引用类型的互换：

#### NSDictionary <–> Object

在上节中，我们把JSContext的globalObject转换成OC对象，发现是NSDictionary类型。要搞清楚这个转换，首先我们对JS这门语言面向对象的特性进行一个简单的了解。在JS中，对象就是一个引用类型的实例。与我们熟悉的OC、Java不一样，对象并不是一个类的实例，因为在JS中并不存在类的概念。ECMA把对象定义为：无序属性的集合，其属性可以包含基本值、对象或者函数。从这个定义我们可以发现，JS中的对象就是无序的键值对，这和OC中的NSDictionary，Java中的HashMap何其相似。

```objc
var person = { name: "Nicholas",age: 17};//JS中的person对象
	NSDictionary *person = @{@"name":@"Nicholas",@"age":@17};//OC中的person dictionary
```

在上面的实例代码中，笔者使用了类似的方式创建了JS中的对象（在JS中叫“对象字面量”表示法）与OC中的NSDictionary，相信可以更有助理解这两个转换。

#### NSBlock <–> Function Object

在上节的例子中，笔者在JSContext赋值了一个”globalFunc”的Block，并可以在JS代码中当成一个函数直接调用。我还可以使用”typeof”关键字来判断globalFunc在JS中的类型：

```objc
NSString *type = [[context evaluateScript:@"typeof globalFunc"] toString];//type的值为"function"
```

通过这个例子，我们也能发现传入的Block对象在JS中已经被转成了”function”类型。”Function Object”这个概念对于我们写惯传统面向对象语言的开发者来说，可能会比较晦涩。而实际上，JS这门语言，除了基本类型以外，就是引用类型。函数实际上也是一个”Function”类型的对象，每个函数名实则是指向一个函数对象的引用。比如我们可以这样在JS中定义一个函数：

```objc
var sum = function(num1,num2){
		return num1 + num2; 
	}
```

同时我们还可以这样定义一个函数（不推荐）:

```objc
var sum = new Function("num1","num2","return num1 + num2");
```

按照第二种写法，我们就能很直观的理解到函数也是对象，它的构造函数就是Function，函数名只是指向这个对象的指针。而NSBlock是一个包裹了函数指针的类，JSCore把Function Object转成NSBlock对象，可以说是很合适的。

### JSExport

> 实现JSExport协议可以开放OC类和它们的实例方法，类方法，以及属性给JS调用。

除了上一节提到的几种特殊类型的转换，我们还剩下NSDate类型，与id、class类型的转换需要弄清楚。而NSDate类型无需赘述，所以我们在这一节重点要弄清楚后两者的转换。

而通常情况下，我们如果想在JS环境中使用OC中的类和对象，需要它们实现JSExport协议，来确定暴露给JS环境中的属性和方法。比如我们需要向JS环境中暴露一个Person的类与获取名字的方法：

```objc
@protocol PersonProtocol <JSExport>
- (NSString *)fullName;//fullName用来拼接firstName和lastName，并返回全名
@end

@interface JSExportPerson : NSObject <PersonProtocol>
  
- (NSString *)sayFullName;//sayFullName方法

@property (nonatomic, copy) NSString *firstName;
@property (nonatomic, copy) NSString *lastName;

@end
```

然后，我们可以把一个JSExportPerson的一个实例传入JSContext，并且可以直接执行fullName方法：

```objc
JSExportPerson *person = [[JSExportPerson alloc] init];
    context[@"person"] = person;
    person.firstName = @"Di";
    person.lastName =@"Tang";
    [context evaluateScript:@"log(person.fullName())"];//调Native方法，打印出person实例的全名
    [context evaluateScript:@"person.sayFullName())"];//提示TypeError，'person.sayFullName' is undefined
```

这就是一个很简单的使用JSExport的例子，但请注意，我们只能调用在该对象在JSExport中开放出去的方法，如果并未开放出去，如上例中的”sayFullName”方法，直接调用则会报TypeError错误，因为该方法在JS环境中并未被定义。

讲完JSExport的具体使用方法，我们来看看我们最开始的问题。当一个OC对象传入JS环境之后，会转成一个JSWrapperObject。那问题来了，什么是JSWrapperObject？在JSCore的源码中，我们可以找到一些线索。首先在JSCore的JSValue中，我们可以发现这样一个方法:

```objc
@method
@abstract Create a JSValue by converting an Objective-C object.
@discussion The resulting JSValue retains the provided Objective-C object.
@param value The Objective-C object to be converted.
@result The new JSValue.
*/
+ (JSValue *)valueWithObject:(id)value inContext:(JSContext *)context;
```

这个API可以传入任意一个类型的OC对象，然后返回一个持有该OC对象的JSValue。那这个过程肯定涉及到OC对象到JS对象的互换，所以我们只要分析一下这个方法的源码（基于 [这个分支](https://svn.webkit.org/repository/webkit/tags/Safari-538.11.1/) 进行分析）。由于源码实现过长，我们只需要关注核心代码，在JSContext中有一个”wrapperForObjCObject”方法，而实际上它又是调用了JSWrapperMap的”jsWrapperForObject”方法，这个方法就可以解答所有的疑惑：

```objc
//接受一个入参object,并返回一个JSValue
- (JSValue *)jsWrapperForObject:(id)object
{
    //对于每个对象，有专门的jsWrapper
    JSC::JSObject* jsWrapper = m_cachedJSWrappers.get(object);
    if (jsWrapper)
        return [JSValue valueWithJSValueRef:toRef(jsWrapper) inContext:m_context];
    JSValue *wrapper;
    //如果该对象是个类对象，则会直接拿到classInfo的constructor为实际的Value
    if (class_isMetaClass(object_getClass(object)))
        wrapper = [[self classInfoForClass:(Class)object] constructor];
    else {
        //对于普通的实例对象，由对应的classInfo负责生成相应JSWrappper同时retain对应的OC对象，并设置相应的Prototype
        JSObjCClassInfo* classInfo = [self classInfoForClass:[object class]];
        wrapper = [classInfo wrapperForObject:object];
    }
    JSC::ExecState* exec = toJS([m_context JSGlobalContextRef]);
    //将wrapper的值写入JS环境
    jsWrapper = toJS(exec, valueInternalValue(wrapper)).toObject(exec);
    //缓存object的wrapper对象
    m_cachedJSWrappers.set(object, jsWrapper);
    return wrapper;
}
```

在我们创建”JSWrapperObject”的对象过程中，我们会通过JSWrapperMap来为每个传入的对象创建对应的JSObjCClassInfo。这是一个非常重要的类，它有这个类对应JS对象的原型（Prototype）与构造函数（Constructor）。然后由JSObjCClassInfo去生成具体OC对象的JSWrapper对象，这个JSWrapper对象中就有一个JS对象所需要的所有信息（即Prototype和Constructor）以及对应OC对象的指针。之后，把这个jsWrapper对象写入JS环境中，即可在JS环境中使用这个对象了。这也就是”JSWrapperObject”的真面目。而我们上文中提到，如果传入的是类，那么在JS环境中会生成constructor对象，那么这点也很容易从源码中看到，当检测到传入的是类的时候（类本身也是个对象），则会直接返回constructor属性，这也就是”constructor object”的真面目，实际上就是一个构造函数。

那现在还有两个问题，第一个问题是，OC对象有自己的继承关系，那么在JS环境中如何描述这个继承关系？第二个问题是，JSExport的方法和属性，又是如何让JS环境中调用的呢？

我们先看第一个问题，继承关系要如何解决？在JS中，继承是通过原型链来实现，那什么是原型呢？原型对象是一个普通对象，而且就是构造函数的一个实例。所有通过该构造函数生成的对象都共享这一个对象，当查找某个对象的属性值，结果不存在时，这时就会去对象的原型对象继续找寻，是否存在该属性，这样就达到了一个封装的目的。我们通过一个Person原型对象快速了解：

```javascript
//原型对象是一个普通对象，而且就是Person构造函数的一个实例。所有Person构造函数的实例都共享这一个原型对象。
Person.prototype = {
   name:  'tony stark',
   age: 48,
   job: 'Iron Man',
   sayName: function() {
     alert(this.name);
   }
}
```

而原型链就是JS中实现继承的关键，它的本质就是重写构造函数的原型对象，链接另一个构造函数的原型对象。这样查找某个对象的属性，会沿着这条原型链一直查找下去，从而达到继承的目的。我们通过一个例子快速了解一下：

```javascript
function mammal (){}
 	mammal.prototype.commonness = function(){
   		alert('哺乳动物都用肺呼吸');
 	}; 

	function Person() {}
	Person.prototype = new mammal();//原型链的生成,Person的实例也可以访问commonness属性了
	Person.prototype.name = 'tony stark';
	Person.prototype.age  = 48;
	Person.prototype.job  = 'Iron Man';
	Person.prototype.sayName = function() {
  		alert(this.name);
	}

	var person1 = new Person();
	person1.commonness(); // 弹出'哺乳动物都用肺呼吸'
	person1.sayName(); // 'tony stark'
```

而我们在生成对象的classinfo的时候（具体代码见”allocateConstructorAndPrototypeWithSuperClassInfo”），还会生成父类的classInfo。对每个实现过JSExport的OC类，JSContext里都会提供一个prototype。比如NSObject类，在JS里面就会有对应的Object Prototype。对于其它的OC类，会创建对应的Prototype，这个prototype的内部属性[Prototype]会指向为这个OC类的父类创建的Prototype。这个JS原型链就能反应出对应OC类的继承关系，在上例中，Person.prototype被赋值为一个mammal的实例对象，即原型的链接过程。

讲完第一个问题，我们再来看看第二个问题。那JSExport是如何暴露OC方法到JS环境的呢？这个问题的答案同样出现在我们生成对象的classInfo的时候：

```objc
Protocol *exportProtocol = getJSExportProtocol();
        forEachProtocolImplementingProtocol(m_class, exportProtocol, ^(Protocol *protocol){
            copyPrototypeProperties(m_context, m_class, protocol, prototype);
            copyMethodsToObject(m_context, m_class, protocol, NO, constructor);
        });
```

对于每个声明在JSExport里的属性和方法，classInfo会在prototype和constructor里面存入对应的property和method。之后我们就可以通过具体的methodName和PropertyName生成的setter和getter方法，来获取实际的SEL。最后就可以让JSExport中的方法和属性得到正确的访问。所以简单点讲，JSExport就是负责把这些方法打个标，以methodName为key，SEL为value，存入一个map（prototype和constructor本质上就是一个Map）中去，之后就可以通过methodName拿到对应的SEL进行调用。这也就解释了上例中，我们调用一个没有在JSExport中开放的方法会显示undefined，因为生成的对象里根本没有这个key。

## 总结

JSCore给iOS App提供了JS可以解释执行的运行环境与资源。对于我们实际开发而言，最主要的就是JSContext和JSValue这两个类。JSContext提供互相调用的接口，JSValue为这个互相调用提供数据类型的桥接转换。让JS可以执行Native方法，并让Native回调JS，反之亦然。

![[3ba5f2e8.png]]

利用JSCore，我们可以做很多有想象空间的事。所有基于JSCore的Hybrid开发基本就是靠上图的原理来实现互相调用，区别只是具体的实现方式和用途不大相同。大道至简，只要正确理解这个基本流程，其它的所有方案不过是一些变通，都可以很快掌握。

## 一些引申阅读

### JSPatch的对象和方法没有实现JSExport协议，JS是如何调OC方法的？

JS调OC并不是通过JSExport。通过JSExport实现的方式有诸多问题，我们需要先写好Native的类，并实现JSExport协议，这个本身就不能满足“Patch”的需求。

所以JSPatch另辟蹊径，使用了OC的Runtime消息转发机制做这个事情，如下面这一个简单的JSPatch调用代码：

```javascript
require('UIView') 
var view = UIView.alloc().init()
```

1. require在全局作用域里生成UIView变量，来表示这个对象是一个OCClass。
2. 通过正则把.alloc()改成._c(‘alloc’)，来进行方法收口，最终会调用_methodFunc()把类名、对象、MethodName通过在Context早已定义好的Native方法，传给OC环境。
3. 最终调用OC的CallSelector方法，底层通过从JS环境拿到的类名、方法名、对象之后，通过NSInvocation实现动态调用。

JSPatch的通信并没有通过JSExport协议，而是借助JSCore的Context与JSCore的类型转换和OC的消息转发机制来完成动态调用，实现思路真的很巧妙。

### 桥方法的实现是怎么通过JSCore交互的？

市面上常见的桥方法调用有两种：

1. 通过UIWebView的delegate方法：shouldStartLoadWithRequest来处理桥接JS请求。JSRequest会带上methodName，通过WebViewBridge类调用该method。执行完之后，会使用WebView来执行JS的回调方法，当然实际上也是调用的WebView中的JSContext来执行JS，完成整个调用回调流程。
2. 通过UIWebView的delegate方法：在webViewDidFinishLoadwebViewDidFinishLoad里通过KVC的方式获取UIWebView的JSContext，然后通过这个JSContext设置已经准备好的桥方法供JS环境调用。

## 参考资料

## 作者简介

* 唐笛，美团点评高级工程师。2017年加入原美团，目前作为外卖iOS团队主力开发，主要负责移动端基础设施建设，动态化等方向相关推进工作，致力于提升移动端研发效率与研发质量。

## 招聘

美团外卖长期招聘Android、iOS、FE 高级/资深工程师和技术专家，base 北京、上海、成都，欢迎有兴趣的同学投递简历到chenhang03#meituan.com。

[深入理解JSCore](https://tech.meituan.com/2018/08/23/deep-understanding-of-jscore.html)