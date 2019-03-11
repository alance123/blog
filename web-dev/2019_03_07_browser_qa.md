# 关于浏览器以及js runtime的一些问题总结

## v8引擎和js runtime什么关系

### V8引擎
V8引擎应该是我们听到最多的引擎，它使用在Chrome浏览器和Node.js中。找到一个解释：A JavaScript engine is a program responsible for translating source code into machine code and executing the translation result on a computer's central processing unit(CPU).

常见的引擎有以下：
* Chrome V8 : Google Chrome中使用的引擎。使用C++实现的。Opera,NodeJS和Couchbase都是使用的它。
* SpiderMonkey: FireFox使用
* Nitro: Safari使用
* Chakra Edge使用

### js runtime
JS Runtime就是运行着JS引擎的环境，它提供接口给开发者调用。有一些我们经常用到的API，比如setTimeout其实都是属于window的，跟V8引擎没有任何关系。因此诸如DOM，Ajax，setTimeout等方法都是由浏览器提供并实现的，这些可以称为Web APIs。

### 浏览器内核
浏览器内核主要分成两部分：渲染引擎和js引擎。渲染引擎主要是负责HTML、CSS以及其他一些东西的渲染，而JS引擎则主要负责对JavaScript的渲染。但是现在JS引擎越来越独立的，基本上所说的内核大多不包含JS引擎了。

| 浏览器     | 内核             |                        js引擎                               |
| ----------|------------------|-------------------------------------------------------------|
|IE -> Edge |trident->EdgeHTML |JScript(IE3.0-IE8.0)/Chakra(IE9+之后）                        |
|Chrome     |webkit->blink     |V8                                                           |
|Safari     |webkit            |Nitro(4-)                                                    |
|Firefox    |Gecko             |SpiderMonkey(1.0-3.0)/TraceMonkey(3.5-3.6)/JaegerMonkey(4.0-)|
|Opera      |Presto->blink     |Linear A(4.0-6.1)/Linera B(7.0-9.2)/Carakan(10.5-)           |
至于移动端，大部分使用的均为webkit这一套，基本兼容webkit就OK。

## 渲染引擎和JS引擎的关系
![渲染引擎和JS引擎关系](http://mayun.itc.cmbchina.cn/uploads/images/2019/0311/112712_af105213_2213.jpeg "render&js_engine.Jpg")
![一张网页的历程](http://mayun.itc.cmbchina.cn/uploads/images/2019/0311/112741_2539be6f_2213.jpeg "render_process.Jpg")

## microtask和macrotask
* 一个事件循环(event loop)会有一个或多个任务队列(task queue)，task queue就是macrotask queue
* 每一个event loop都有一个microtask queue
* task queue == macrotask queue ！= microtask queue
* 一个任务task可以放入macrotask queue 也可以放入microtask queue中
* 当一个task被放入队列queue（macro或micro)，那这个task就可以被立即执行了

整个eventloop基本可以总结为
1. 在macrotask队列中执行最早的那个task，然后移出
2. 执行microtask队列中所有可用的任务，然后移出
3. 下一个循环，执行下一个macrotask中的任务（再跳到第二步）

### 摘抄一段解释
一个事件循环有一个或者多个任务队列（task queues）。任务队列是task的有序列表，这些task是以下工作的对应算法：`Events(比如下面的例子中的click事件的触发就是Task)`，Parsing，Callbacks，Using a resource，Reacting to DOM manipulation,setImmediate(setImmediate.js), MessageChannel,window.postMessage，setTimeout,setInterval[等等](https://segmentfault.com/a/1190000008589736)。

每一个任务都来自一个特定的`任务源`（task source）。所有来自一个特定任务源并且属于特定事件循环的任务，通常必须被加入到同一个任务队列中，但是来自不同任务源的任务可能会放在不同的任务队列中。

举个例子，用户代理有一个处理鼠标和键盘事件的任务队列。用户代理可以给这个队列比其他队列多3/4的执行时间，以确保交互的响应而不让其他任务队列饿死（starving），并且不会乱序处理任何一个任务队列的事件。

`每个事件循环`都有一个进入[microtask](https://www.w3.org/TR/html5/webappapis.html#microtask)检查点（performing a microtask checkpoint）的flag标志，这个标志初始为false。它被用来组织反复调用‘进入microtask检查点’的算法。总结一下，一个事件循环里有很多个任务队列（task queues）来自不同任务源，每一个任务队列里的任务是严格按照先进先出的顺序执行的，但是不同任务队列的任务的执行顺序是不确定的。按我的理解就是，浏览器会自己调度不同任务队列。网上很多文章会提到macrotask这个概念，其实就是指代了标准里阐述的task。

标准同时还提到了microtask的概念，也就是微任务。看一下标准阐述的事件循环的进程模型：
<pre>
1.选择当前要执行的任务队列，选择一个最先进入任务队列的任务，如果`没有任务可以选择(如果任务已经执行完毕，直接跳转到microtasks步骤，否则执行第2步)`，则会跳转至microtask的执行步骤。
2.将事件循环的当前运行任务设置为已选择的任务。
3.运行任务。
4.将事件循环的当前运行任务设置为null。
5.将运行完的任务从任务队列中移除。
6.microtasks步骤：进入microtask检查点（performing a microtask checkpoint ）。
7.更新界面渲染。
8.返回第一步。   
</pre>

执行进入microtask检查点时，用户代理会执行以下步骤：
<pre>
1.设置进入microtask检查点的标志为true。
2.当事件循环的微任务队列不为空时：选择一个最先进入microtask队列的microtask；设置事件循环的当前运行任务为已选择的microtask；运行microtask；设置事件循环的当前运行任务为null；将运行结束的microtask从microtask队列中移除。
3.对于相应事件循环的每个环境设置对象（environment settings object）,通知它们哪些promise为rejected。
4.清理indexedDB的事务。
5.设置进入microtask检查点的标志为false。    
</pre>


现在我们知道了。在事件循环中，用户代理会不断从`task队列`中按顺序取task执行，每执行完一个task都会检查microtask队列是否为空（执行完一个task的具体标志是`函数执行栈`为空），如果不为空则会一次性执行完所有microtask。然后再进入下一个循环去task队列中取下一个task执行...

那么哪些行为属于task或者microtask呢？标准没有阐述，但各种技术文章总结都如下：
<pre>
macrotasks: script(整体代码), setTimeout, setInterval, setImmediate, I/O, UI rendering
microtasks: process.nextTick, Promises, Object.observe(废弃), MutationObserver
</pre>

关于事件循环还要注意一点,你可以查看[为什么Vue使用MutationObserver做批量处理？](https://github.com/liangklfangl/react-article-bucket/blob/master/vue/inner-core-concept.md#2%E4%B8%BA%E4%BB%80%E4%B9%88vue%E4%BD%BF%E7%94%A8mutationobserver%E5%81%9A%E6%89%B9%E9%87%8F%E5%A4%84%E7%90%86),即:每次macrotask执行完成后会进行UI更新，所以**为了使得界面更快更新，我们应该使用microtask而不是macrotask**。


## V8是怎么工作的
在V8的5.9版本之前，它拥有两个编译器：
* full-codegen —— 一个简单且快速的编译器，可以生成简单但相对较慢的机器码
* Grankshaft —— 一个较复杂的即时优化编译器，可以生成高度优化后的代码

V8引擎内部的多线程分工：
* 主线程完成你所希望运行的主要内容：获取你书写的代码，编译执行
* 还有一个单独的线程用于编译，以便于主线程可以继续执行，而前者可以在同时进行代码优化
* Profiler线程，它将通知运行时那些方法花费大量时间，以便于Grankshaft可以优化他们
* 一些线程来处理垃圾收集器的扫码

首次执行JavaScript代码，V8利用full-codegen直接将解析后的JavaScript不经过任何优化，直接编译为机器码。这使它可以非常快速地开始执行机器码。注意，V8不使用中间字节码，因此不需要解释器。当代码运行一段时间后，profiler线程以及手机来足够的数据来通知运行时应该优化哪个方法。然后，Grankshaft在另一个线程开始优化。它将JavaScript抽象语法树转换为名为Hydrogen的高级静态单元分配表示（SSA），并尝试去优化这个Hydrogen图。大多数优化在这个层级完成。

### V8引擎所做的优化
[V8引擎中5个优化代码的技巧](https://lyn-ho.github.io/posts/4d26265b/)