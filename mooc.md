# React源码解析，先人一步窥探未来

## 先来打个基础

#### 准备工作

* 下载源码
* 把我看的时候的源码上传到单独仓库，让想保持同步的同学下载这个
* 源码和跑demo的代码有区别，一个是编译前的ES6，一个是打包后的ES5
* 创建一个文档记录所有的数据类型

#### React和React-DOM的关系

我们平时都说自己用React搭建应用，但是有多少同学知道React本身源码才1000多行，而平时我们可能只会用一次`ReactDOM.render`的`react-dom`的源码有两万多行。那他们之间到底有什么关系，又为什么要分成两个包呢？

#### JSX to Javascript

我们写React应用都是用的JSX，但显然代码在运行的时候肯定是Javascript，那么JSX转换成Javascript之后是什么样子的呢？

#### ReactElement

什么是`ReactElement`，有什么特点和作用，有那些不同类型的`ReactElement`
·
#### Component

`ClassComponent`继承`Component`或者`PureComponent`，查看源码。

#### createRef

ref的三种不同的使用方式，以及为什么要用`createRef`

#### 新老context演示

#### ReactDOM入口API一览



## 新功能展示

#### new lifecyles

#### AsyncMode、defferedUpdate、flushSync

#### supense

#### lazy component


## 会引起更新的操作和原理

* 每个操作的流程
* 创建update和updateQueue
* 到进入scheduler为止

#### Fiber是什么

Fiber其实是帮助React把一次更新从整颗树需要进行操作，细化到每一个节点需要进行的操作，因为**一个整体的更新被细化到了一组小范围的更新，就可以方便于控制这次更新的进度，是否可以中断等。**

#### ReactDOM.render 和 hydrate

#### update & updateQueue

#### expirationTime的计算

#### setState

#### forceUpdate



## scheduler

按照Sync来讲，跳过`expirationTime`

#### 总体流程图解

* 加入到`scheduleRoot`
* 通过react-scheduler调度异步任务
* 同步Fiber树work处理流程
* 异步Fiber树work处理流程

#### scheduleWork

#### requestWork

#### performWork

#### performWorkOnRoot

#### renderRoot




## beginWork

各类update，一个个讲解

* 忽略新老`context`相关的内容

#### reconciler children

#### key for array and iterator


## completeUnitOfWork

跳过事件



## completeRoot & commitRoot



## features

#### old context

#### new context

#### ref

#### hydrate

#### controlled input

#### error doundaries



## supense 详解

讲解异步的原理，supense的原理，pirority放到这里讲

#### expirationTime & childExpirationTime

#### react-scheduler

#### suspendTime & retry



## events 和 diffDOM initialDom