# Priority

首先我们要知道一点，在React16.4的分支上，这部分代码是不存在的，在React发布了16.5版本之后，我切换到了master分支的代码，这部分代码才实现。那么这部分代码是干嘛的呢？答案是`sequense`，在17年末到18年初，FB的Dan就四处进行分享，主要内容就是会在React17版本中发布的一些新功能，一个就是`AsyncRendering`，还有一个就是`sequense`。

### sequense

首先来讲一下什么是`sequense`，他提供了React在`render`方法中进行异步处理的功能，比如如下的例子

```js
const DataReander = () => {
  const data = SomeProvider.get('dataId')

  return (
    <div>{data.info}</div>
  )
}

ReactDOM.render(
  <Placehodler fallback={'something render able'}>
    <DataReander />
  </Placeholder>,
  document.getElementById('app')
)
```

`SomeProvider.get('dataId')`这里是一个异步获取数据的操作，按照正常的逻辑，React的`render`方法是同步渲染的，所以在渲染的时候`data`肯定还是不存在的，一般来说获取数据会返回一个`Promise`。

那么React如何处理这个情况呢？`SomeProvider.get('dataId')`会`throw`这个`promise`，或者你可以让这个方法返回一个`promise`，然后你判断他是否是`promise`，并自己`throw`，比如：

```js
const DataReander = () => {
  const data = SomeProvider.get('dataId')

  if (data is promise) { // 伪代码
    throw data
  }

  return (
    <div>{data.info}</div>
  )
}
```

这是第一个React引入这个概念想要解决的问题

### Async Rendering

如果仅仅是因为`sequense`那么还不需要引入这么复杂的概念，因为渲染都是同步的，只需要考虑到`promise`的`resolve`就可以了。但是如果结合上`Async Rendering`，就变得很复杂。

先来说一下`Async Rendering`的概念，在进行节点非常多的`React`树的渲染过程中，因为任务非常复杂，可能导致消耗的时间很长，因为JS是单线程的，如果`React`一次`setState`导致非常多的节点需要重新计算并渲染，那么可能会影响页面动画的渲染效果，导致卡帧的现象。

所以React提供了这个功能，这个功能告诉我们这次`setState`的优先级不高，如果执行时间较长，可以中间中断一下，等渲染引擎有空了再来执行我。而这个能实现的核心就是`requestIdleCallback`，React会给异步任务一个计时，比如`33`毫秒的`30`帧稳定帧时间，在`33`毫秒内如果任务没执行完会被中断，把执行权给浏览器去执行动画什么之类的，然后把任务放到`requestIdleCallback`的回调里面，等有空了再来执行。

而为了防止这个任务永远处于没空执行的状态，React会给每次任务生成的时候设置一个过期时间。根据优先级的不同，会有不同的过期时间

1. 同步任务优先级最高，所以过期时间是`1`，也就是直接执行不考虑中断的情况
2. 异步模式下的`InteractiveUpdates`过期时间是`22`毫秒，这类操作是响应事件的操作，所以优先级较高
3. 一步模式，502毫秒

另外开发模式下`InteractiveUpdates`的过期时间会稍微长一些，52毫秒，让开发者可以更容易发现卡顿现象。

所以如果结合了`Async Rendering`和`sequense`，在异步模式下出现了中断的任务，那么这时候就要考虑这些时间了：

1. `placehodler`的`delayMs`设置的超时时间
2. Promise的`resolve`的时间
3. 在`Promise`正在pending的情况下有新的任务进来的时间

同时还存在一个问题，那就是任务过期之后会强制执行这个任务，那么有那么多不同的等待时间在，应该根据哪个来判断任务是否执行呢？

**这就是React引入这个概念的原因**


# Priority的实现

在React中引入了一系列时间记录参数来判断最终的过期时间应该是哪个，我们先来看一下参数

1. `earliestPendingTime`
2. `latestPendingTime`
3. `earliestSuspendedTime`
4. `latestSuspendedTime`
5. `latestPingedTime`

首先解释几个概念

##### pending

`pending`是指目前正在执行的任务的过期时间，在一个新的任务被创建的时候，会通过`markPendingPriorityLevel`进行记录，React只会记录最早的和最老的过期时间。

##### suspended

`suspended`是挂起的意思，对应的也就是`suquense`的情况，在一个任务被挂起的时候，会记录这个挂起任务的过期时间。类似`pending`，他也只记录最早和最老的

##### pinged

在挂起任务的Promise resolve的时候，会判断当时设置的过期时间是否处于目前整个应用的过期时间之间。如果是的话这个时间会被设置为`pingedTime`，如果不在这个区间内，则会重新计算任务的过期时间，并按照节点是否是异步渲染的模式被安排到更新队列当中。**相当于执行了一次新的`setState`**



