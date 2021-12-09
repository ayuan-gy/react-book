# React是如何调度任务的

首先我们要知道哪些操作会让React开始一次任务调度

1. `ReactDOM.render`或者`ReactDOM.hydrate`
2. `this.setState`
3. `this.forceUpdate`
4. suspend component promise resolve or reject

那么从这几个方法入手，我们来看一下

### ReactDOM.render

首先会创建`ReactRoot`对象，然后调用他的`render`方法

```js
ReactRoot.prototype.render = function(
  children: ReactNodeList,
  callback: ?() => mixed,
): Work {
  const root = this._internalRoot;
  const work = new ReactWork();
  callback = callback === undefined ? null : callback;
  if (__DEV__) {
    warnOnInvalidCallback(callback, 'render');
  }
  if (callback !== null) {
    work.then(callback);
  }
  DOMRenderer.updateContainer(children, root, null, work._onCommit);
  return work;
};
```

其中`DOMRenderer`是`react-reconciler/src/ReactFiberReconciler`，他的`updateContainer`如下在这里计算了一个时间，这个时间叫做`expirationTime`，顾名思义就是这次更新的 **超时时间**。

关于时间是如何计算的[看这里](./time.md)

然后调用了`updateContainerAtExpirationTime`，在这个方法里调用了`scheduleRootUpdate`就非常重要了

```js
export function updateContainer(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  callback: ?Function,
): ExpirationTime {
  const current = container.current;
  const currentTime = requestCurrentTime();
  const expirationTime = computeExpirationForFiber(currentTime, current);
  return updateContainerAtExpirationTime(
    element,
    container,
    parentComponent,
    expirationTime,
    callback,
  );
}

export function updateContainerAtExpirationTime(
  element: ReactNodeList,
  container: OpaqueRoot,
  parentComponent: ?React$Component<any, any>,
  expirationTime: ExpirationTime,
  callback: ?Function,
) {
  // TODO: If this is a nested container, this won't be the root.
  const current = container.current;
  const context = getContextForSubtree(parentComponent);
  if (container.context === null) {
    container.context = context;
  } else {
    container.pendingContext = context;
  }

  return scheduleRootUpdate(current, element, expirationTime, callback);
}
```

{% method %}

# 开始调度

首先要生成一个`update`，不管你是`setState`还是`ReactDOM.render`造成的React更新，都会生成一个叫`update`的对象，并且会赋值给`Fiber.updateQueue`

关于`update`请[看这里](./update.md)

然后就是调用`scheduleWork`。注意到这里之前`setState`和`ReactDOM.render`是不一样，但进入`schedulerWork`之后，就是任务调度的事情了，跟之前你是怎么调用的没有任何关系

{% sample lang="js" %}

```js
function scheduleRootUpdate(
  current: Fiber,
  element: ReactNodeList,
  expirationTime: ExpirationTime,
  callback: ?Function,
) {
  const update = createUpdate(expirationTime);

  update.payload = {element};

  callback = callback === undefined ? null : callback;
  if (callback !== null) {
    warningWithoutStack(
      typeof callback === 'function',
      'render(...): Expected the last optional `callback` argument to be a ' +
        'function. Instead received: %s.',
      callback,
    );
    update.callback = callback;
  }
  enqueueUpdate(current, update);

  scheduleWork(current, expirationTime);
  return expirationTime;
}
```

{% endmethod %}

{% method %}

# scheduleWork

这里先`scheduleWorkToRoot`，这一步非常重要，他主要做了一下几个任务

* 找到当前`Fiber`的root
* 给更新节点的父节点链上的每个节点的`expirationTime`设置为这个`update`的`expirationTime`，除非他本身时间要小于`expirationTime`
* 给更新节点的父节点链上的每个节点的`childExpirationTime`设置为这个`update`的`expirationTime`，除非他本身时间要小于`expirationTime`

最终返回root节点的`Fiber`对象

然后进入一个判断：`!isWorking && nextRenderExpirationTime !== NoWork && expirationTime < nextRenderExpirationTime`，我们来解释一下这几个变量的意思

1. `isWorking`代表是否正在工作，在开始`renderRoot`和`commitRoot`的时候会设置为true，也就是在`render`和`commit`两个阶段都会为`true`
2. `nextRenderExpirationTime`在是新的`renderRoot`的时候会被设置为当前任务的`expirationTime`，而且一旦他被，只有当下次任务是`NoWork`的时候他才会被再次设置为`NoWork`，当然最开始也是`NoWork`

那么这个条件就很明显了：**目前没有任何任务在执行，并且之前有执行过任务，同时当前的任务比之前执行的任务过期时间要早（也就是优先级要高）**

那么这种情况会出现在什么时候呢？答案就是：**上一个任务是异步任务（优先级很低，超时时间是502ms），并且在上一个时间片（初始是33ms）任务没有执行完，而且等待下一次`requestIdleCallback`的时候新的任务进来了，并且超时时间很短（52ms或者22ms甚至是Sync），那么优先级就变成了先执行当前任务，也就意味着上一个任务被打断了（interrupted）**

被打断的任务会从当前节点开始往上推出`context`，因为在React只有一个`stack`，而下一个任务会从头开始的，所以在开始之前需要清空之前任务的的`stack`。

[`context`请看这里](./context.md)

[`unwindWork`请看这里](./unwindWork.md)

然后重置所有的公共变量：

```js
nextRoot = null;
nextRenderExpirationTime = NoWork;
nextLatestAbsoluteTimeoutMs = -1;
nextRenderDidError = false;
nextUnitOfWork = null;
```

##### markPendingPriorityLevel

TODO: PriorityLevel有很多的时间变量，不知道具体的含义是什么

```js
if (
  !isWorking ||
  isCommitting ||
  nextRoot !== root
)
```
这个判断条件就比较简单了，`!isWorking || isCommitting`简单来说就是要么处于没有work的状态，要么只能在render节点，不能处于commit阶段（比较好奇什么时候会在commit阶段有新的任务进来，commit都是同步的无法打断）。还有一个选项`nextRoot !== root`，这个的意思就是你的APP如果有两个不同的root，这时候也符合条件。

在符合条件之后就`requestWork`了

{% sample lang="js" %}

```js
function scheduleWork(fiber: Fiber, expirationTime: ExpirationTime) {
  const root = scheduleWorkToRoot(fiber, expirationTime);

  if (enableSchedulerTracing) {
    storeInteractionsForExpirationTime(root, expirationTime, true);
  }

  if (
    !isWorking &&
    nextRenderExpirationTime !== NoWork &&
    expirationTime < nextRenderExpirationTime
  ) {
    // This is an interruption. (Used for performance tracking.)
    interruptedBy = fiber;
    resetStack();
  }
  markPendingPriorityLevel(root, expirationTime);
  if (
    // If we're in the render phase, we don't need to schedule this root
    // for an update, because we'll do it before we exit...
    !isWorking ||
    isCommitting ||
    // ...unless this is a different root than the one we're rendering.
    nextRoot !== root
  ) {
    const rootExpirationTime = root.expirationTime;
    requestWork(root, rootExpirationTime);
  }
}

function scheduleWorkToRoot(fiber: Fiber, expirationTime): FiberRoot | null {
  // Update the source fiber's expiration time
  if (
    fiber.expirationTime === NoWork ||
    fiber.expirationTime > expirationTime
  ) {
    fiber.expirationTime = expirationTime;
  }
  let alternate = fiber.alternate;
  if (
    alternate !== null &&
    (alternate.expirationTime === NoWork ||
      alternate.expirationTime > expirationTime)
  ) {
    alternate.expirationTime = expirationTime;
  }
  // Walk the parent path to the root and update the child expiration time.
  let node = fiber.return;
  if (node === null && fiber.tag === HostRoot) {
    return fiber.stateNode;
  }
  while (node !== null) {
    alternate = node.alternate;
    if (
      node.childExpirationTime === NoWork ||
      node.childExpirationTime > expirationTime
    ) {
      node.childExpirationTime = expirationTime;
      if (
        alternate !== null &&
        (alternate.childExpirationTime === NoWork ||
          alternate.childExpirationTime > expirationTime)
      ) {
        alternate.childExpirationTime = expirationTime;
      }
    } else if (
      alternate !== null &&
      (alternate.childExpirationTime === NoWork ||
        alternate.childExpirationTime > expirationTime)
    ) {
      alternate.childExpirationTime = expirationTime;
    }
    if (node.return === null && node.tag === HostRoot) {
      return node.stateNode;
    }
    node = node.return;
  }
  return null;
}

function resetStack() {
  if (nextUnitOfWork !== null) {
    let interruptedWork = nextUnitOfWork.return;
    while (interruptedWork !== null) {
      unwindInterruptedWork(interruptedWork);
      interruptedWork = interruptedWork.return;
    }
  }

  nextRoot = null;
  nextRenderExpirationTime = NoWork;
  nextLatestAbsoluteTimeoutMs = -1;
  nextRenderDidError = false;
  nextUnitOfWork = null;
}
```

{% endmethod %}


{% method %}

# requestWork

`addRootToSchedule`把root加入到调度队列，但是要注意一点，不会存在两个相同的root前后出现在队列中

```js
function addRootToSchedule(root: FiberRoot, expirationTime: ExpirationTime) {
  // Add the root to the schedule.
  // Check if this root is already part of the schedule.
  if (root.nextScheduledRoot === null) {
    // This root is not already scheduled. Add it.
    root.expirationTime = expirationTime;
    if (lastScheduledRoot === null) {
      firstScheduledRoot = lastScheduledRoot = root;
      root.nextScheduledRoot = root;
    } else {
      lastScheduledRoot.nextScheduledRoot = root;
      lastScheduledRoot = root;
      lastScheduledRoot.nextScheduledRoot = firstScheduledRoot;
    }
  } else {
    // This root is already scheduled, but its priority may have increased.
    const remainingExpirationTime = root.expirationTime;
    if (
      remainingExpirationTime === NoWork ||
      expirationTime < remainingExpirationTime
    ) {
      // Update the priority.
      root.expirationTime = expirationTime;
    }
  }
}
```

可以看出来，如果第一次调用`addRootToSchedule`的时候，`nextScheduledRoot`是`null`，这时候公共变量`firstScheduledRoot`和`lastScheduledRoot`也是`null`，所以会把他们都赋值成`root`，同时`root.nextScheduledRoot = root`。然后第二次进来的时候，如果前后`root`是同一个，那么之前的`firstScheduledRoot`和`lastScheduledRoot`都是root，所以`lastScheduledRoot.nextScheduledRoot = root`就等于`root.nextScheduledRoot = root`

这么做是因为同一个`root`不需要存在两个，因为前一次调度如果中途被打断，下一次调度进入还是从同一个`root`开始，就会把新的任务一起执行了。

之后根据`expirationTime`调用`performSyncWork`还是`scheduleCallbackWithExpirationTime`

`scheduleCallbackWithExpirationTime`是根据时间片来执行任务的，会涉及到`requestIdleCallback`，详细解析看[这里](./react-scheduler.md)

`isBatchingUpdates`和`isUnbatchingUpdates`涉及到事件系统，看[React事件系统](../event-system/README.md)

他们最终都要调用`performWork`

{% sample lang="js" %}

```js
function requestWork(root: FiberRoot, expirationTime: ExpirationTime) {
  addRootToSchedule(root, expirationTime);
  if (isRendering) {
    // Prevent reentrancy. Remaining work will be scheduled at the end of
    // the currently rendering batch.
    return;
  }

  if (isBatchingUpdates) {
    // Flush work at the end of the batch.
    if (isUnbatchingUpdates) {
      // ...unless we're inside unbatchedUpdates, in which case we should
      // flush it now.
      nextFlushedRoot = root;
      nextFlushedExpirationTime = Sync;
      performWorkOnRoot(root, Sync, true);
    }
    return;
  }

  // TODO: Get rid of Sync and use current time?
  if (expirationTime === Sync) {
    performSyncWork();
  } else {
    scheduleCallbackWithExpirationTime(root, expirationTime);
  }
}
```

{% endmethod %}

{% method %}

# performWork

这里判断是否有`deadline`来分成两种渲染方式，但最大的差距其实是`while`循环的判断条件，有`deadline`的多了一个条件`(!deadlineDidExpire || currentRendererTime >= nextFlushedExpirationTime)`

我们先来看相同的部分

```js
nextFlushedRoot !== null &&
nextFlushedExpirationTime !== NoWork &&
(minExpirationTime === NoWork ||
  minExpirationTime >= nextFlushedExpirationTime)
```

下一个输出节点不是`null`，并且过期时间不是`NoWork`，同时*超时时间是`NoWork`，或者超时时间大于下个节点的超时时间*

{% sample lang="js" %}

```js
function performWork(minExpirationTime: ExpirationTime, dl: Deadline | null) {
  deadline = dl;

  // Keep working on roots until there's no more work, or until we reach
  // the deadline.
  findHighestPriorityRoot();

  if (deadline !== null) {
    recomputeCurrentRendererTime();
    currentSchedulerTime = currentRendererTime;

    if (enableUserTimingAPI) {
      const didExpire = nextFlushedExpirationTime < currentRendererTime;
      const timeout = expirationTimeToMs(nextFlushedExpirationTime);
      stopRequestCallbackTimer(didExpire, timeout);
    }

    while (
      nextFlushedRoot !== null &&
      nextFlushedExpirationTime !== NoWork &&
      (minExpirationTime === NoWork ||
        minExpirationTime >= nextFlushedExpirationTime) &&
      (!deadlineDidExpire || currentRendererTime >= nextFlushedExpirationTime)
    ) {
      performWorkOnRoot(
        nextFlushedRoot,
        nextFlushedExpirationTime,
        currentRendererTime >= nextFlushedExpirationTime,
      );
      findHighestPriorityRoot();
      recomputeCurrentRendererTime();
      currentSchedulerTime = currentRendererTime;
    }
  } else {
    while (
      nextFlushedRoot !== null &&
      nextFlushedExpirationTime !== NoWork &&
      (minExpirationTime === NoWork ||
        minExpirationTime >= nextFlushedExpirationTime)
    ) {
      performWorkOnRoot(nextFlushedRoot, nextFlushedExpirationTime, true);
      findHighestPriorityRoot();
    }
  }

  // We're done flushing work. Either we ran out of time in this callback,
  // or there's no more work left with sufficient priority.

  // If we're inside a callback, set this to false since we just completed it.
  if (deadline !== null) {
    callbackExpirationTime = NoWork;
    callbackID = null;
  }
  // If there's work left over, schedule a new callback.
  if (nextFlushedExpirationTime !== NoWork) {
    scheduleCallbackWithExpirationTime(
      ((nextFlushedRoot: any): FiberRoot),
      nextFlushedExpirationTime,
    );
  }

  // Clean-up.
  deadline = null;
  deadlineDidExpire = false;

  finishRendering();
}
```

{% endmethod %}