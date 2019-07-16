# Eventloop

<a name="ml9cmm"></a>
## [](#ml9cmm)什么是 Eventloop
eventloop 其实就是 js 代码的运行机制。在了解机制原理之前，我们需要先理解几个概念

<a name="r0odmo"></a>
#### [](#r0odmo)线程

1. 线程和进程, 从本质上来说，两个名词都是 CPU **工作时间片**的一个描述。

2. 进程描述了 CPU 在**运行指令及加载和保存上下文所需的时间**，放在应用上来说就代表了一个程序。线程是进程中的更小单位，描述了执行一段指令所需的时间。

3. JS 是**单线程**执行的


<a name="uqykil"></a>
#### [](#uqykil)执行栈

1. 可以把执行栈认为是一个存储函数调用的**栈结构**，遵循先进后出的原则。


<a name="1cw9mf"></a>
#### [](#1cw9mf)任务队列

1. 存放任务的队列，遵循先进先出的原则

2. 在 Eventloop 中，任务源可以分为 **微任务**（microtask） 和 **宏任务**（macrotask）。在 ES6 规范中，microtask 称为 `jobs`，macrotask 称为 `task`。


<a name="te4odb"></a>
## [](#te4odb)原理
JS 代码其实就是往**执行栈**里面放入函数。当遇到异步的代码时，会被挂起并在需要执行的时候加入到**任务队列**中。一旦执行栈为空，Eventloop 就会从任务队列中拿出需要执行的代码并放入执行栈中执行，**所以本质上来说 JS 中的异步还是同步行为**。<br />前面提到，任务源可以分为微任务和宏任务，那么到底哪些属于微任务哪些属于宏任务呢？

- 微任务包括 `process.nextTick` ，`promise.then`，`MutationObserver`。

- 宏任务包括 `script` ， `setTimeout` ，`setInterval` ，`setImmediate` ，`I/O` ，`UI rendering`。


所以 Eventloop 执行顺序如下所示：

- 首先执行同步代码(即主进程里的任务)，这属于宏任务

- 当执行完所有同步代码后，执行栈为空，查询是否有异步代码需要执行

- 执行所有微任务

- 当执行完所有微任务后，从宏任务队列里取出第一个宏任务放进主进程执行

- 然后开始下一轮 Eventloop


![](https://cdn.nlark.com/lark/0/2018/png/20513/1545818395811-6e61df76-f86e-450d-9265-376a4ec1d032.png#width=747)


