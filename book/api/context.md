# Stack

在 React 的更新过程中有一个`Stack`模块，在遍历每个节点的过程中，会扮演存储上下文的一个角色，他会存储很多信息，让我们来一一分解讲解。

首先我们来讲一下`Stack`的构成

```js
const valueStack: Array<any> = []

let index = -1

function createCursor<T>(defaultValue: T): StackCursor<T> {
  return {
    current: defaultValue,
  }
}

function isEmpty(): boolean {
  return index === -1
}

function pop<T>(cursor: StackCursor<T>, fiber: Fiber): void {
  if (index < 0) {
    return
  }

  cursor.current = valueStack[index]

  valueStack[index] = null

  index--
}

function push<T>(cursor: StackCursor<T>, value: T, fiber: Fiber): void {
  index++

  valueStack[index] = cursor.current

  cursor.current = value
}
```

精简之后的核心模块代码如上，所有内容会存储在`valueStack`这个数组上，总体的操作都非常简单，我相信大家啊都能看得懂。

而在这个唯一的数组上要存储以下不怎么相关的数据：

- `childContext`
- `NewContext`
- `HostContext`
- `HostContainer`

为了能够在同一个栈中区分不同的功能，React 设计了一个`StackCursor`类型，用来区分不同的类型的数据的当前值。对于`cursor`的操作很简单，入栈的时候设置`cursor.current`为新的值，出栈的时候设置`cursor.current`为上一个值

但是这里还存在一个问题，那就是数据入栈的顺序，如果入栈位置出栈的时候用的`cursor`不同，就会导致数据错乱。React 中防止出现这个问题的方式，是通过**每个节点在 beginWork 的时候入栈，在 completeUnitOfWork 的时候出栈，严格按照遍历树的顺序**

# childContext

`childContext`是 React 的遗留 API，在 17 版本中会被移除，该 API 对于 React 渲染的效率影响很大，不推荐继续使用。目前 React 源码中调用该 API 相关的方法会加上`Legacy`字样。

```js
let contextStackCursor: StackCursor<Object> = createCursor(emptyContextObject)
let didPerformWorkStackCursor: StackCursor<boolean> = createCursor(false)
```

这里存在三个`cursor`

### contextStackCursor

记录当前组件和他的父树一起提供给子树的`childContext`对象，初始默认是`emptyContextObject {}`。

对`FiberRoot`会执行第一次`push`，除非自行调用`renderSubtreeIntoContainer`，不然`root`的`context`都是`{}`，除了初次渲染，`push`的值都是`false`，表明目前`context`没有变化。

```js
// updateHostRoot的时候会调用
function pushTopLevelContextObject(
  fiber: Fiber,
  context: Object,
  didChange: boolean,
): void {
  push(contextStackCursor, context, fiber)
  push(didPerformWorkStackCursor, didChange, fiber)
}
```

之后只有`ClassComponent`能够提供`childContext`，在`updateClassComponent`的过程中会调用`pushContextProvider`来推入新的子树`context`对象。

```js
function pushContextProvider(workInProgress: Fiber): boolean {
  const instance = workInProgress.stateNode
  const memoizedMergedChildContext =
    (instance && instance.__reactInternalMemoizedMergedChildContext) ||
    emptyContextObject

  previousContext = contextStackCursor.current
  push(contextStackCursor, memoizedMergedChildContext, workInProgress)
  push(
    didPerformWorkStackCursor,
    didPerformWorkStackCursor.current,
    workInProgress,
  )

  return true
}
```

可以看到这里只是从`instance.__reactInternalMemoizedMergedChildContext`读取对象，但是在`updateClassComponent`调用这个方法的时候并没有计算出新的`state`，所以是否有新的`context`也是未知。注意这里给全局变量`previousContext`赋值了`contextStackCursor.current`，所以他是当前组件的父树提供的`context`的集合。在后续`finishClassComponent`的时候如果`state`或者`props`有更新，那么需要重新计算`context`，会执行`invalidateContextProvider`

```js
function invalidateContextProvider(
  workInProgress: Fiber,
  type: any,
  didChange: boolean,
): void {
  const instance = workInProgress.stateNode

  if (didChange) {
    const mergedContext = processChildContext(
      workInProgress,
      type,
      previousContext,
    )
    instance.__reactInternalMemoizedMergedChildContext = mergedContext

    pop(didPerformWorkStackCursor, workInProgress)
    pop(contextStackCursor, workInProgress)
    push(contextStackCursor, mergedContext, workInProgress)
    push(didPerformWorkStackCursor, didChange, workInProgress)
  } else {
    pop(didPerformWorkStackCursor, workInProgress)
    push(didPerformWorkStackCursor, didChange, workInProgress)
  }
}
```

`processChildContext`会计算出新的`childContext`，然后赋值给`__reactInternalMemoizedMergedChildContext`，并且之前对于当前组件已经`push`过一次了，所以这里要先`pop`再`push`，而且两个`cursor`的顺序要调换。而如果新老`context`都没有变化，会设置`didPerformWorkStackCursor`为`false`，可以优化子树，不需要执行不必要的更新。**注意这里的`didChange`跟`shouldComponentUpdate`有关，另外需要注意`PureComponent`不会判断`context`是否变化。**

### didPerformWorkStackCursor

这个就是用来标记子树的`context`是否变化的，在以上的代码中已经很明显了，所以就不再分析。

```js
function hasContextChanged(): boolean {
  return didPerformWorkStackCursor.current
}
```

# NewContext

这个是新的`Context API`。

```js
const valueCursor: StackCursor<mixed> = createCursor(null)
```

相对来说他的逻辑会简单挺多，`Provider`处理如下：

```js
export function pushProvider<T>(providerFiber: Fiber, nextValue: T): void {
  const context: ReactContext<T> = providerFiber.type._context
  if (isPrimaryRenderer) {
    push(valueCursor, context._currentValue, providerFiber)
    context._currentValue = nextValue
  } else {
    push(valueCursor, context._currentValue2, providerFiber)
    context._currentValue2 = nextValue
  }
}

export function popProvider(providerFiber: Fiber): void {
  const currentValue = valueCursor.current
  pop(valueCursor, providerFiber)
  const context: ReactContext<any> = providerFiber.type._context
  if (isPrimaryRenderer) {
    context._currentValue = currentValue
  } else {
    context._currentValue2 = currentValue
  }
}
```

ReactDOM 中`isPrimaryRenderer`为`true`，注意这里设置了`context._currentValue`，在后续要用到`context`的时候，比如通过`Consumer`读取值，那么只需要读取这个值就可以了。

在`React16.6`之后呢增加了`ClassComponent.contextType`快速订阅新 API 的方式，还有`hooks`中`functionalComponent`可以使用`useContext`订阅新 API，所以呢这里有两个方法来处理这个逻辑：

- `propagateContextChange`
- `readContext`

代码有点长，用个 gist 来展示吧

<script src="https://gist.github.com/Jokcy/c6b0e36383dbfc0e0e254f2ffe416a6b.js"></script>

`propagateContextChange`在`ContextProvider`检测到`context`的值有变化的情况下调用，他会遍历他的子树，找有`firstContextDependency`属性的`fiber`，并检测他是否有以来当前的`ContextProvider`，如果有的话会在这个组件上创建一个更新，并且这个更新的`expirationTime`是当前正在执行的更新的`expirationTime`，也就是说在这个渲染周期肯定会被渲染，因为创建了更新，所以肯定要更新`expirationTime`同时还要更新父链上的`childExpirationTime`。

那么`firstContextDependency`哪里来的呢？就是在组件在调用`readContext`的时候，目前能看的源码（16.6）还没有`hooks`的源码，所以目前能看到的是`ClassCompnent.contextType`，对于这种情况他的依赖只会有一个，而`hooks`是可以读取多个的。

`observedBits`在目前的源码中没看到什么作用，虽然源码中`ContextConsumer`中可以使用`unstable_observedBits`这个属性，但是目前没有任何说明这是用来干嘛的。_猜测跟`hooks`有关？_

# HostContext & HostContainer

```js
let contextStackCursor: StackCursor<HostContext | NoContextT> = createCursor(
  NO_CONTEXT,
)
let contextFiberStackCursor: StackCursor<Fiber | NoContextT> = createCursor(
  NO_CONTEXT,
)
let rootInstanceStackCursor: StackCursor<Container | NoContextT> = createCursor(
  NO_CONTEXT,
)
```

这里主管的内容有：

### pushHostContainer

对于`HostRoot`和`Proatl`他们会有挂载节点，所以会有`container`

### pushHostContext

这个是用来对原生节点进行入栈的操作，主要记录的是`NameSpace`

```js
const HTML_NAMESPACE = 'http://www.w3.org/1999/xhtml'
const MATH_NAMESPACE = 'http://www.w3.org/1998/Math/MathML'
const SVG_NAMESPACE = 'http://www.w3.org/2000/svg'

switch (type) {
  case 'svg':
    return SVG_NAMESPACE
  case 'math':
    return MATH_NAMESPACE
  default:
    return HTML_NAMESPACE
}
```
