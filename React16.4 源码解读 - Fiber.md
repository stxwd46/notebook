# React16.4 源码解读 - Fiber

> 写该文章之前，我查阅了 React 源码的相关资料。对于一些概念，如果其他作者已经给出详细易懂的解释，本文会直接引用，并在下方的「参考资料」中给出相关链接。同时会记录一些自己阅读源码后的理解。有不对的地方，欢迎指正。


<a name="3heiwb"></a>
## [](#3heiwb)Reconciler 与 Renderer
在了解 Fiber 之前，我们先了解一下 React 里当一个组件更新的时候内部的流程。<br />当一个组件需要更新( 比如调用了 setState )时，React 内部会分为两个阶段( phase )执行组件更新。
<a name="kfmvso"></a>
#### [](#kfmvso)phase 1 —— Reconciler
Reconciler 其实就是 React 用来比较两棵树的差异从而决定哪些部分需要更新的算法。他有个更为人所知的名字，叫 **Virual DOM.**<br />`componentWillMount` 、 `componentWillUpdate` 、 `shouldComponentWillUpdate` 、 `componentWillReceiveProps` 等生命周期方法在这个阶段被调用执行
<a name="aafeyf"></a>
#### [](#aafeyf)phase 2 —— Renderer
Renderer 是将 Reconciler 阶段对比出来后新的树真正更新到渲染层上。<br />`componentDidMount` 、 `componentDidUpdate` 、 ` ComponentWillUnmount`  等生命周期方法在这个阶段被调用执行

<a name="1983380b"></a>
## [](#Fiber是什么)Fiber是什么
React 源码中对 Fiber 的解释是这样的

```javascript
// packages/react-reconciler/src/ReactFiber.js
// A Fiber is work on a Component that needs to be done or was done. There can
// be more than one per component.
```
翻译过来就是 「Fiber 工作在需要去完成或者已经完成的组件上。一个组件可以有一个或多个 Fiber」。

单从注释上看可能还没法很清晰的看出 Fiber 是什么。我们可以先阅读一下关于 Fiber 结构的一篇[文档](https://github.com/acdlite/react-fiber-architecture)<br />这篇文档里有几个要点

1. A fiber represents a **unit of work**.

2. Fiber reimplements the reconciler. It is not principally concerned with rendering, though renderers will need to change to support (and take advantage of) the new architecture.

3. pause work and come back to it later.

4. assign priority to different types of work.

5. reuse previously completed work.

6. abort work if it's no longer needed.



也就是说，Fiber 重构了 Reconciler 这个阶段，让它得以不再关注渲染相关的事情(渲染交给 Renderer 阶段执行)，它可以实现异步渲染，赋予不同任务不同优先级，优先级高的优先渲染；同时它可以复用之前已经完成的任务并中止不再需要的任务，优化渲染性能。**每一个 Fiber 代表一个工作单元**。这时候再来看看上面的注释，也就大致能明白，每一个 Fiber 都对应着一个真实页面中的 React 组件,  并负责维护该组件的生命周期。

<a name="xaebau"></a>
## [](#xaebau)Fiber 数据结构
Fiber 其实是一个对象，这里列出一些比较重要的 Fiber 属性。了解熟悉这些属性，对之后阅读 `setState` 的源码会很有帮助，setState 源码分析也是我接下来准备写的另一篇文章。

```javascript
// packages/react-reconciler/src/ReactFiber.js

export type Fiber = {|
  tag: TypeOfWork, // fiber 的类型
  key:  null | string, // 该子节点的唯一标识
  stateNode: any, // 该 fiber 对应的真实 dom 节点
  return: Fibe | null, // 指向 fiber 树中的父节点
  child: Fiber | null, // 指向第一个子节点
  sibling: Fiber | null, // 指向兄弟节点
  memoizedState: any, // 输出的 state
  effectTag: TypeOfSideEffect, // side effect类型
  nextEffect: Fiber | null, // 单链表结构，快速找到下一个具有 side-effects 的 fiber
  firstEffect: Fiber | null, // 指向第一个具有 side-effects的 fiber
  lastEffect: Fiber | null, // 指向最后一个具有 side-effects的 fiber
  expirationTime: ExpirationTime, // 该 fiber 的渲染优先级
  alternate: Fiber | null, // fiber 在 update 的时候克隆出来的一个新的 fiber
|}
```

从源码上看， Fiber 是在 render 函数里被创建的。我们在 render 函数中创建的 React Element 树在第一次渲染的时候会创建一棵结构一模一样的Fiber节点树。不同的React Element类型对应不同的Fiber节点类型。一个React Element的工作就由它对应的Fiber节点来负责。

<a name="c0r1it"></a>
## [](#c0r1it)Virtual DOM
Fiber 是 Virtual DOM 比较两棵树差异算法中的工作单元，它在重构后的 Reconciler 发挥着核心的作用

1. render 函数中创建的 React Element 树，在第一次渲染的时候会创建一棵结构一模一样的 Fiber 节点树

2. update 的时候，会从原来的 Fiber（**current**）clone出一个新的 Fiber（**alternate, 即流程图里的 workInProgress，后面所说的 alternate 均指流程图里的 workInProgress**）。两个 Fiber diff出的变化（side effect）记录在 **alternate** 上

3. 在更新结束后 **alternate** 会取代之前的 **current** 的成为新的 **current** 节点，从而完成组件更新


这就是 Virtual Dom 的原理，流程图如下

![](https://cdn.nlark.com/lark/0/2018/png/20513/1539851212135-39c6cdb6-1097-4458-af7c-7fbd80243186.png#width=827)

这也就很好的解释了源码注释中所说的「一个组件可以有一个或多个 Fiber」，当它刚创建的时候，只有一个 Fiber（**current**），当该组件处于更新状态的时候，有两个 Fiber ( **current** 和 **alternate** )。

<a name="gmncnu"></a>
## [](#gmncnu)Fiber 的形态
Fiber 一共有两种形态，即 **current** 和 **alternate**，在这里有必要详细的解释一下他们的[概念](https://github.com/acdlite/react-fiber-architecture#alternate)
<a name="98gebt"></a>
#### [](#98gebt)1. current
current 比较容易理解，就是根据  React Element 树创建的一棵结构一模一样的 Fiber 节点树
<a name="pkfryc"></a>
#### [](#pkfryc)2. alternate
alternate 是组件更新的时候，从 current Fiber 克隆出来的一个新 Fiber，它有两种阶段
<a name="gsmmnb"></a>
##### [](#gsmmnb)work-in-progress
未完成态，即尚未完成更新的 fiber
<a name="d7ydwo"></a>
##### [](#d7ydwo)flush
完成态，即将输出渲染到屏幕

current 的 alternate 是 work-in-progress, work-in-progress 的 alternate 是 current。

<a name="dvbmnm"></a>
## [](#dvbmnm)Fiber 的类型
上文有给出 Fiber 的结构，里面有个 tag 属性，value 值为 `TypeOfWork` ，代表着 Fiber 的类型，所以我们来看一下 `TypeOfWork` 里一共有哪几种类型

```javascript
// packages/shared/ReactTypeOfWork.js

export const IndeterminateComponent = 0; // Before we know whether it is functional or class
export const FunctionalComponent = 1;
export const ClassComponent = 2;
export const HostRoot = 3; // Root of a host tree. Could be nested inside another node.
export const HostPortal = 4; // A subtree. Could be an entry point to a different renderer.
export const HostComponent = 5;
export const HostText = 6;
export const CallComponent_UNUSED = 7;
export const CallHandlerPhase_UNUSED = 8;
export const ReturnComponent_UNUSED = 9;
export const Fragment = 10;
export const Mode = 11;
export const ContextConsumer = 12;
export const ContextProvider = 13;
export const ForwardRef = 14;
export const Profiler = 15;
export const PlaceholderComponent = 16;
```

常用的类型有以下几种
<a name="364uot"></a>
#### [](#364uot)FunctionalComponent
函数组件，比如
```javascript
const FunctionalComponent = ({label}) => <div>{label}</div>;
```

<a name="na32rr"></a>
#### [](#na32rr)**ClassComponent**
这种就是我们最常用的应用层面的React组件。ClassComponent 是一个继承自 React.Component 的类的实例。比如
```javascript
class ClassComponent extends React.Component {
  render() {
    return <block>{this.props.label}</block>;
  }
}
```

<a name="6rh8xt"></a>
#### [](#6rh8xt)**HostRoot**
ReactDOM.render() 时的根节点。可以被嵌套在其他节点里

<a name="d3rpdx"></a>
#### [](#d3rpdx)**HostComponent**
React中最常见的抽象节点，是 ClassComponent 的组成部分。具体的实现取决于 React 运行的平台。在浏览器环境下就代表DOM 节点，可以理解为所谓的虚拟 DOM 节点。HostComponent中的 Host 就代表这种组件的具体操作逻辑是由Host环境注入的。

<a name="80itvy"></a>
## [](#80itvy)effectTag
上文给出的 Fiber 结构里面还有个 effectTag 属性，用来定义 side-effects 的类型。那么 side-effects 是用来做什么的呢？<br />**当更新结束后的新的 Fiber 树，准备开始渲染到 UI 上的时候，便是根据 side-effects 来决定对相应的 DOM 节点进行什么操作。**<br />**所以 effectTag 便是可对 dom 节点进行的操作动作对应的值，并且以二进制位表示**

```javascript
export type TypeOfSideEffect = number;

// Don't change these two values. They're used by React Dev Tools.
export const NoEffect = /*              */ 0b00000000000; // 0，没有更新内容，和原 fiber 树一致，不需要重新渲染
export const PerformedWork = /*         */ 0b00000000001; // 1，有更新内容，需要重新渲染 

// You can change the rest (and add more).
export const Placement = /*             */ 0b00000000010; // 2，插入
export const Update = /*                */ 0b00000000100; // 4，更新
export const PlacementAndUpdate = /*    */ 0b00000000110; // 6，插入并更新
export const Deletion = /*              */ 0b00000001000; // 8，删除
export const ContentReset = /*          */ 0b00000010000; // 16，重置
export const Callback = /*              */ 0b00000100000; // 32
export const DidCapture = /*            */ 0b00001000000; // 64，异常
export const Ref = /*                   */ 0b00010000000; // 128
export const Snapshot = /*              */ 0b00100000000; // 256

// Update & Callback & Ref & Snapshot
export const LifecycleEffectMask = /*   */ 0b00110100100; // 420, 对于还未完成更新操作但已经准备要渲染到 UI 层的 fiber，用该标志告知不需要执行他的生命周期

// Union of all host effects
export const HostEffectMask = /*        */ 0b00111111111; // 511

export const Incomplete = /*            */ 0b01000000000; // 512
export const ShouldCapture = /*         */ 0b10000000000; // 1024
```

<a name="oa9hvo"></a>
## [](#oa9hvo)**参考资料**

1. [http://zxc0328.github.io/2017/09/28/react-16-source/](http://zxc0328.github.io/2017/09/28/react-16-source/)

2. [https://github.com/UNDERCOVERj/tech-blog/issues/12](https://github.com/UNDERCOVERj/tech-blog/issues/12)

3. [https://juejin.im/post/5b87590051882542ff3e6c0b](https://juejin.im/post/5b87590051882542ff3e6c0b)

4. [https://github.com/acdlite/react-fiber-architecture](https://github.com/acdlite/react-fiber-architecture#alternate)

5. [https://www.youtube.com/watch?v=ZCuYPiUIONs](https://www.youtube.com/watch?v=ZCuYPiUIONs)



