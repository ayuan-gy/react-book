# Placeholder

`Placeholder`是React17即将发布的`sequense`功能中的一部分，他是一个组件，用来包裹`sequense`并提供`sequense`组件在`resolve`之前可以显示一些加载效果

`Placeholder`提供两个（目前已知）参数，`fallback`和`delayMs`，前者是在`sequense`组件`resolve`值钱显示的*正在加载中效果*，后者是用来告诉`Placeholder`如果`sequense`组件在`delayMs`时间之前就`resolve`了，就不需要显示*正在加载中效果*

那么`Placehodler`组件是如何实现的呢？

{% method %}

# updatePlaceholderComponent

我们来看一下他的源码

在React更新的过程中，如果遇到`Placeholder`组件，首先判断其是否已经捕获到了`error`，`nextDidTimeout`的结果有两种方式：

1. `current !== null && workInProgress.updateQueue !== null`的时候永远为`true`
2. 等于`!alreadyCaptured`

**事实上经过测试，目前非正式版没有发现第一种情况**，而且根据源码，也没有发现任何情况下会为`Placeholder`组件生成`updateQueue`的时候，**猜测可能后面会加入`delayMs`的逻辑**

所以基本上都会等于`!alreadyCaptured`，那么`alreadyCaptured`怎么来的呢？

那就是他包裹的`sequense`组件`throw Promise`的时候，React会通过`throwException`方法向上检索`sequense`组件的父组件，并找到`Placeholder`组件，然后给其增加`ShouldCapture`的`effectTag`，然后执行`completeUnitOfWork`，因为`throw Promise`导致`sequense`组件是`Incomplete`的，所以会执行`unwindWork`逻辑，在向上检索的时候，遇到`Placehodler`组件，因为他有`ShouldCapture`的`side effect`，所以会执行以下逻辑：

```js
function unwindWork() {
  switch(fiber.tag) {
    // ...
    case PlaceholderComponent: {
      const effectTag = workInProgress.effectTag;
      if (effectTag & ShouldCapture) {
        workInProgress.effectTag = (effectTag & ~ShouldCapture) | DidCapture;
        return workInProgress;
      }
      return null;
    }
    // ...
  }
}
```

会`return`当前`workInProgress`，并赋值给`completeUnitOfWork`里的参数`next`，在`completeUnitOfWork`的逻辑里，如果`next`存在，说明在`completeWork`的时候产生了新的任务，就需要对当前`workInProgress`从新执行`beginWork`，这时候`Placeholder`就带着`DidCapture`的`side effect`再次进入`updatePlaceholderComponent`，于是`nextDidTimeout`就变成了`true`，然后他的`children`就是`fallback`了。

关于`beginWork`和`completeWork`的逻辑可以看这里

* [`beginWork`](../flow/begin-work.md)
* ['completeWork](../flow/completeWork.md)

{% sample lang="js" %}

```js
function updatePlaceholderComponent(
  current,
  workInProgress,
  renderExpirationTime,
) {
  if (enableSuspense) {
    const nextProps = workInProgress.pendingProps;

    // Check if we already attempted to render the normal state. If we did,
    // and we timed out, render the placeholder state.
    const alreadyCaptured =
      (workInProgress.effectTag & DidCapture) === NoEffect;

    let nextDidTimeout;
    if (current !== null && workInProgress.updateQueue !== null) {
      // We're outside strict mode. Something inside this Placeholder boundary
      // suspended during the last commit. Switch to the placholder.
      workInProgress.updateQueue = null;
      nextDidTimeout = true;
      // If we're recovering from an error, reconcile twice: first to delete
      // all the existing children.
      reconcileChildren(current, workInProgress, null, renderExpirationTime);
      current.child = null;
      // Now we can continue reconciling like normal. This has the effect of
      // remounting all children regardless of whether their their
      // identity matches.
    } else {
      nextDidTimeout = !alreadyCaptured;
    }

    if ((workInProgress.mode & StrictMode) !== NoEffect) {
      if (nextDidTimeout) {
        // If the timed-out view commits, schedule an update effect to record
        // the committed time.
        workInProgress.effectTag |= Update;
      } else {
        // The state node points to the time at which placeholder timed out.
        // We can clear it once we switch back to the normal children.
        workInProgress.stateNode = null;
      }
    }

    // If the `children` prop is a function, treat it like a render prop.
    // TODO: This is temporary until we finalize a lower level API.
    const children = nextProps.children;
    let nextChildren;
    if (typeof children === 'function') {
      nextChildren = children(nextDidTimeout);
    } else {
      nextChildren = nextDidTimeout ? nextProps.fallback : children;
    }

    workInProgress.memoizedProps = nextProps;
    workInProgress.memoizedState = nextDidTimeout;
    reconcileChildren(
      current,
      workInProgress,
      nextChildren,
      renderExpirationTime,
    );
    return workInProgress.child;
  } else {
    return null;
  }
}
```

{% endmethod %}