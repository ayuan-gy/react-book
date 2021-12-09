# 从源码看Hooks

React 17-alpha中新增了新功能：`Hooks`。总结他的功能就是：让`FunctionalComponent`具有`ClassComponent`的功能。

```js
import React, { useState, useEffect } from 'react'

function FunComp(props) {
  const [data, setData] = useStat('initialState')

  function handleChange(e) {
    setData(e.target.value)
  }

  useEffect(() => {
    subscribeToSomething()

    return () => {
      unSubscribeToSomething()
    }
  })

  return (
    <input value={data} onChange={handleChange} />
  )
}
```

按照*Dan*的说法，设计`Hooks`主要是解决`ClassComponent`的几个问题：

1. 很难复用逻辑（只能用HOC，或者render props），会导致组件树层级很深
2. 会产生巨大的组件（指很多代码必须写在类里面）
3. 类组件很难理解，比如方法需要`bind`，`this`指向不明确

这些确实是存在的问题，比如我们如果用了`react-router`+`redux`+`material-ui`，很可能随便一个组件最后`export`出去的代码是酱紫的：

```js
export default withStyle(style)(connect(/*something*/)(withRouter(MyComponent)))
```
这就是一个4层嵌套的`HOC`组件

同时，如果你的组件内事件多，那么你的`constructor`里面可能会酱紫：

```js
class MyComponent extends React.Component {
  constructor() {
    // initiallize

    this.handler1 = this.handler1.bind(this)
    this.handler2 = this.handler2.bind(this)
    this.handler3 = this.handler3.bind(this)
    this.handler4 = this.handler4.bind(this)
    this.handler5 = this.handler5.bind(this)
    // ...more

  }
}
```

虽然最新的`class`语法可以用`handler = () => {}`来快捷绑定，但也就解决了一个声明的问题，整体的复杂度还是在的。

然后还有在`componentDidMount`和`componentDidUpdate`中订阅内容，还需要在`componentWillUnmount`中取消订阅的代码，里面会存在很多重复性工作。最重要的是，在一个`ClassComponent`中的生命周期方法中的代码，是很难在其他组件中复用的，这就导致了了代码复用率低的问题。

还有就是`class`代码对于打包工具来说，很难被压缩，比如方法名称。

更多详细的大家可以去看[`ReactConf`的视频](https://www.youtube.com/watch?v=V-QO-KO90iQ&t=3060s)，我这里就不多讲了，**这篇文章的主题是从源码的角度讲讲`Hooks`是如何实现的**


### 先来了解一些基础概念

首先`useState`是一个方法，它本身是无法存储状态的

其次，他运行在`FunctionalComponent`里面，本身也是无法保存状态的

`useState`只接收一个参数`initial value`，并看不出有什么特殊的地方。所以React在一次重新渲染的时候如何获取之前更新过的`state`呢？

在开始讲解源码之前，大家先要建立一些概念：

###### React Element

`JSX`翻译过来之后是`React.createElement`，他最终返回的是一个`ReactElement`对象，他的数据解构如下：

```js
const element = {
  $$typeof: REACT_ELEMENT_TYPE, // 是否是普通Element_Type

  // Built-in properties that belong on the element
  type: type,  // 我们的组件，比如`class MyComponent`
  key: key,
  ref: ref,
  props: props,

  // Record the component responsible for creating this element.
  _owner: owner,
};
```

这其中需要注意的是`type`，在我们写`<MyClassComponent {...props} />`的时候，他的值就是`MyClassComponent`这个`class`，而不是他的实例，实例是在后续渲染的过程中创建的。

###### Fiber

每个节点都会有一个对应的`Fiber`对象，他的数据解构如下：

```js
function FiberNode(
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,
) {
  // Instance
  this.tag = tag;
  this.key = key;
  this.elementType = null;  // 就是ReactElement的`$$typeof`
  this.type = null;         // 就是ReactElement的type
  this.stateNode = null;

  // Fiber
  this.return = null;
  this.child = null;
  this.sibling = null;
  this.index = 0;

  this.ref = null;

  this.pendingProps = pendingProps;
  this.memoizedProps = null;
  this.updateQueue = null;
  this.memoizedState = null;
  this.firstContextDependency = null;

  // ...others
}
```

在这里我们需要注意的是`this.memoizedState`，这个`key`就是用来存储在上次渲染过程中最终获得的节点的`state`的，每次执行`render`方法之前，React会计算出当前组件最新的`state`然后赋值给`class`的实例，再调用`render`。

**所以很多不是很清楚React原理的同学会对React的`ClassComponent`有误解，认为`state`和`lifeCycle`都是自己主动调用的，因为我们继承了`React.Component`，它里面肯定有很多相关逻辑。事实上如果有兴趣可以去看一下`Component`的源码，大概也就是100多行，非常简单。所以在React中，`class`仅仅是一个载体，让我们写组件的时候更容易理解一点，毕竟组件和`class`都是封闭性较强的**

### 开始讲源码

既然我们知道了存储`state`和`props`的是在`Fiber`对象上，那么我们就不需要担心上面提到的`FunctionalComponent`无法存储状态的问题了，所以接下去就让我们一步步看看`useState`具体做了啥吧。

*由于是alpha版本，所以在gitgub上没找到源码，似乎React团队的发布策略是正式版发布才会把代码更新到master分支。所以只能看打包过的源码了*

```js
// react中的useState
function useState(initialState) {
  var dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}

function resolveDispatcher() {
  var dispatcher = ReactCurrentOwner.currentDispatcher;
  !(dispatcher !== null) ? invariant(false, 'Hooks can only be called inside the body of a function component.') : void 0;
  return dispatcher;
}
```

到这里React中的`useState`就结束了，剩下的就是`ReactCurrentOwner.currentDispatcher`是个啥，这个东西会在`react-dom`中渲染的过程中变化，所以我们要到`react-dom`中寻找

在`renderRoot`中我们找到了：

```js
function renderRoot(root, isYieldy, isExpired) {
  // ...
  ReactCurrentOwner$2.currentDispatcher = Dispatcher;

  // workLoop

  ReactCurrentOwner$2.currentDispatcher = null;
  resetHooks();
}

var Dispatcher = {
  readContext: readContext,
  useCallback: useCallback,
  useContext: useContext,
  useEffect: useEffect,
  useImperativeMethods: useImperativeMethods,
  useLayoutEffect: useLayoutEffect,
  useMemo: useMemo,
  useMutationEffect: useMutationEffect,
  useReducer: useReducer,
  useRef: useRef,
  useState: useState
};
```

`renderRoot`这个方法是React开始渲染整棵树的入口，讲起来就超级长了，所以有机会做React源码分析的时候再详细讲，可以关注一波获取最新动态。在这里我们只需要知道，在开始渲染的时候给他设置了`currentDispatcher`，而渲染的过程中是会调用每个节点的，也就是说执行到我们有`Hooks`的组件的时候自然会调用`Dispatcher.useState`

这里还有一个方法叫`resetHooks`，我们后面再讲。

`useState`的代码：

```js
function useState(initialState) {
  return useReducer(basicStateReducer,
  // useReducer has a special case to support lazy useState initializers
  initialState);
}
```

`useReducer`的代码较长，我放在gist里

<script src="https://gist.github.com/Jokcy/de0c7dc7140ec5d7d563a61e103eac25.js"></script>


这里主要做了什么呢？首先要找到`workInProgressHook`，这个就跟`Fiber`跟新的时候的`workInProgress`类似，指代的是当前正在工作的对象。在这里就是执行到了哪个`useState`，要找到他对应的`Hook`对象，数据结构如下：

```js
{
  memoizedState: null,

  baseState: null,
  queue: null,
  baseUpdate: null,

  next: null
};
```

跟`Fiber`对象一样，他也有一个`memoizedState`，**在使用Hooks的时候，我们把`ClassComponent`中以对象存储的`state`，拆成一个个`key`对应的`Hook`的关系，所以每个`Hook`会对应一个`memoizedState`**

找到`workInProgressHook`使用的是`createWorkInProgressHook`，在React中存在着这种`current => workInProgress`的关系，这个一下两下讲不清楚，就不在这详细展开了。这里只需要记住两个关键点：

* 如果第一次渲染，每个`workInProgressHook`都会重新创建
* 如果之前有创建过，会从之前保存的上面复制一个

然后`reducer`会在这个`workInProgressHook`上执行更新，这部分代码也比较复杂，就不详细展开了，直接讲一下重点：

* `queue`是调用`useState`返回的那个方法的时候生成的，我们可能一次性调用很多次，所以`queue`是个链状数据结构
* 




### updateFunctionalComponent

看到这里我们先停一下，我们先来了解一下什么时候才会执行到`useState`，答案就是在`FunctionalComponent`被执行的时候，那么也就是`updateFunctionalComponent`的时候（这个也有机会分析源码的时候再详细讲）。在这里我们看到这么一句代码：

```js
// Component就是FunctionalComponent本身
// nextProps就是新的props
nextChildren = finishHooks(Component, nextProps, nextChildren, context);
```

`finishHooks`代码如下：

<script src="https://gist.github.com/Jokcy/d54b8631ebd40c9d234c262bb2a8becd.js"></script>

`didScheduleRenderPhaseUpdate`在调用`useState`返回的更新方法的时候会设置为`true`，我们先默认他为`true`



# 注意
* 目前`react-hot-loader`不能和`hooks`一起使用，[详情](https://github.com/gaearon/react-hot-loader/issues/1088)
