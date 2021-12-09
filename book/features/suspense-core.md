# Suspense

`suspense`的原理是通过`throw thenable`，然后显示`Suspense`组件的`fallback`，等到`thenable`状态变更之后再根据结果进行渲染。所以核心肯定就是在如何处理抛出的`thenable`对象。

略长的代码，我们可以区分来看，注意这里每个循环向抛出异常的组件的父链上找所有`Suspense`组件。先看第一个`do while`循环，这个循环并不是特别重要，主要是找两个时间值：

- `startTimeMs`
- `earliestTimeoutMs`

前者跟上一次`Suspense`组件被`commit`的时间有关，在`commitAllLifecycle`中被设置。

后者跟我们设置在`Suspense`组件上的属性`maxDuration`有关，大致意思是过多少毫秒之后才真正被提交。

然后进入第二个循环，首先绑定抛出的异常状态变化之后的处理函数`retrySuspendedRoot`。

其次这个循环主要根据`Suspense`组件是否处于`ConcurrentMode`进行不同的处理

#### 没有处于`ConcurrentMode`

这种情况会从简处理，在本次渲染中并不进行异常捕获，直接把异常节点以`null`进行渲染，并且增加`Incomplete`副作用，让后面`completeUnitOfWork`的时候走`unwindWork`的流程。并且如果报错的组件是`ClassComponent`同时是初次渲染，标记组件为`IncompleteClassComponent`，他在后面的更新过程中会特殊处理，主要区别是要删除`current`，完全按照初次渲染进行。

#### 对于处于`ConcurrentMode`

这种情况会走正常`Capture`流程，会增加`ShouldCapture`副作用，然后根据``

{% method %}

{% sample lang="js" %}

```js
function throwException(
  root: FiberRoot,
  returnFiber: Fiber,
  sourceFiber: Fiber,
  value: mixed,
  renderExpirationTime: ExpirationTime,
) {
  sourceFiber.effectTag |= Incomplete
  sourceFiber.firstEffect = sourceFiber.lastEffect = null

  if (
    value !== null &&
    typeof value === 'object' &&
    typeof value.then === 'function'
  ) {
    // This is a thenable.
    const thenable: Thenable = (value: any)

    let workInProgress = returnFiber
    let earliestTimeoutMs = -1
    let startTimeMs = -1
    do {
      if (workInProgress.tag === SuspenseComponent) {
        const current = workInProgress.alternate
        if (current !== null) {
          const currentState: SuspenseState | null = current.memoizedState
          if (currentState !== null && currentState.didTimeout) {
            // Reached a boundary that already timed out. Do not search
            // any further.
            const timedOutAt = currentState.timedOutAt
            startTimeMs = expirationTimeToMs(timedOutAt)
            // Do not search any further.
            break
          }
        }
        let timeoutPropMs = workInProgress.pendingProps.maxDuration
        if (typeof timeoutPropMs === 'number') {
          if (timeoutPropMs <= 0) {
            earliestTimeoutMs = 0
          } else if (
            earliestTimeoutMs === -1 ||
            timeoutPropMs < earliestTimeoutMs
          ) {
            earliestTimeoutMs = timeoutPropMs
          }
        }
      }
      workInProgress = workInProgress.return
    } while (workInProgress !== null)

    // Schedule the nearest Suspense to re-render the timed out view.
    workInProgress = returnFiber
    do {
      if (
        workInProgress.tag === SuspenseComponent &&
        shouldCaptureSuspense(workInProgress.alternate, workInProgress)
      ) {
        const pingTime =
          (workInProgress.mode & ConcurrentMode) === NoEffect
            ? Sync
            : renderExpirationTime

        // Attach a listener to the promise to "ping" the root and retry.
        let onResolveOrReject = retrySuspendedRoot.bind(
          null,
          root,
          workInProgress,
          sourceFiber,
          pingTime,
        )
        if (enableSchedulerTracing) {
          onResolveOrReject = Schedule_tracing_wrap(onResolveOrReject)
        }
        thenable.then(onResolveOrReject, onResolveOrReject)
        if ((workInProgress.mode & ConcurrentMode) === NoEffect) {
          workInProgress.effectTag |= CallbackEffect

          // Unmount the source fiber's children
          const nextChildren = null
          reconcileChildren(
            sourceFiber.alternate,
            sourceFiber,
            nextChildren,
            renderExpirationTime,
          )
          sourceFiber.effectTag &= ~Incomplete

          if (sourceFiber.tag === ClassComponent) {
            sourceFiber.effectTag &= ~LifecycleEffectMask
            const current = sourceFiber.alternate
            if (current === null) {
              sourceFiber.tag = IncompleteClassComponent
            }
          }

          // Exit without suspending.
          return
        }

        let absoluteTimeoutMs
        if (earliestTimeoutMs === -1) {
          absoluteTimeoutMs = maxSigned31BitInt
        } else {
          if (startTimeMs === -1) {
            const earliestExpirationTime = findEarliestOutstandingPriorityLevel(
              root,
              renderExpirationTime,
            )
            const earliestExpirationTimeMs = expirationTimeToMs(
              earliestExpirationTime,
            )
            startTimeMs = earliestExpirationTimeMs - LOW_PRIORITY_EXPIRATION
          }
          absoluteTimeoutMs = startTimeMs + earliestTimeoutMs
        }
        renderDidSuspend(root, absoluteTimeoutMs, renderExpirationTime)

        workInProgress.effectTag |= ShouldCapture
        workInProgress.expirationTime = renderExpirationTime
        return
      }
      // This boundary already captured during this render. Continue to the next
      // boundary.
      workInProgress = workInProgress.return
    } while (workInProgress !== null)
    // No boundary was found. Fallthrough to error mode.
    value = new Error(
      'An update was suspended, but no placeholder UI was provided.',
    )
  }

  // error handle
}
```

{% endmethod %}
