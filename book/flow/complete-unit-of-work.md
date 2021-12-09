{% method %}

# completeUnitOfWork

对于是否具有`Incomplete`副作用的组件会采取不同的措施：

- 有：调用`unwindWork`
- 无：调用`completeWork`

`Incomplete`是在组件渲染过程中有错误被捕获的时候增加

对于一个节点完成工作之后，需要重置父链上所有节点的`childExpirationTime`，调用`resetChildExpirationTime`。

渲染完成之后要赋值父`Fiber`的`effect`链

- 先把他本身的`firstEffect -> lastEffect`链放到父亲的`effect`链的最后
- 如果节点自身也有`side effect`，那么把自己放在父亲的`effect`链的最后。

最后如果有兄弟节点，直接返回兄弟节点，因为要对其执行`update`。如果没有则返回父节点，因为父节点已经执行过更新，也可以`complete`。

{% sample lang="js" %}

```js
function completeUnitOfWork(workInProgress: Fiber): Fiber | null {
  while (true) {
    const current = workInProgress.alternate

    const returnFiber = workInProgress.return
    const siblingFiber = workInProgress.sibling

    if ((workInProgress.effectTag & Incomplete) === NoEffect) {
      // This fiber completed.
      nextUnitOfWork = completeWork(
        current,
        workInProgress,
        nextRenderExpirationTime,
      )
      resetChildExpirationTime(workInProgress, nextRenderExpirationTime)

      if (
        returnFiber !== null &&
        // Do not append effects to parents if a sibling failed to complete
        (returnFiber.effectTag & Incomplete) === NoEffect
      ) {
        if (returnFiber.firstEffect === null) {
          returnFiber.firstEffect = workInProgress.firstEffect
        }
        if (workInProgress.lastEffect !== null) {
          if (returnFiber.lastEffect !== null) {
            returnFiber.lastEffect.nextEffect = workInProgress.firstEffect
          }
          returnFiber.lastEffect = workInProgress.lastEffect
        }

        const effectTag = workInProgress.effectTag
        // Skip both NoWork and PerformedWork tags when creating the effect list.
        // PerformedWork effect is read by React DevTools but shouldn't be committed.
        if (effectTag > PerformedWork) {
          if (returnFiber.lastEffect !== null) {
            returnFiber.lastEffect.nextEffect = workInProgress
          } else {
            returnFiber.firstEffect = workInProgress
          }
          returnFiber.lastEffect = workInProgress
        }
      }

      if (siblingFiber !== null) {
        // If there is more work to do in this returnFiber, do that next.
        return siblingFiber
      } else if (returnFiber !== null) {
        // If there's no more work in this returnFiber. Complete the returnFiber.
        workInProgress = returnFiber
        continue
      } else {
        // We've reached the root.
        return null
      }
    } else {
      if (workInProgress.mode & ProfileMode) {
        // Record the render duration for the fiber that errored.
        stopProfilerTimerIfRunningAndRecordDelta(workInProgress, false)
      }

      const next = unwindWork(workInProgress, nextRenderExpirationTime)
      // Because this fiber did not complete, don't reset its expiration time.
      if (workInProgress.effectTag & DidCapture) {
        // Restarting an error boundary
        stopFailedWorkTimer(workInProgress)
      } else {
        stopWorkTimer(workInProgress)
      }

      if (next !== null) {
        stopWorkTimer(workInProgress)

        // enableProfilerTimer...

        next.effectTag &= HostEffectMask
        return next
      }

      if (returnFiber !== null) {
        // Mark the parent fiber as incomplete and clear its effect list.
        returnFiber.firstEffect = returnFiber.lastEffect = null
        returnFiber.effectTag |= Incomplete
      }

      if (__DEV__ && ReactFiberInstrumentation.debugTool) {
        ReactFiberInstrumentation.debugTool.onCompleteWork(workInProgress)
      }

      if (siblingFiber !== null) {
        // If there is more work to do in this returnFiber, do that next.
        return siblingFiber
      } else if (returnFiber !== null) {
        // If there's no more work in this returnFiber. Complete the returnFiber.
        workInProgress = returnFiber
        continue
      } else {
        return null
      }
    }
  }

  return null
}
```

{% endmethod %}

{% method %}

# resetChildExpirationTime

简单来说这里不仅要看当前节点的`child`的`expirationTime`，还需要看`child`的`childExpirationTime`，并且要从所有兄弟节点中找最小的那个。

过程没啥好讲的，算法挺简单的。

{% sample lang="js" %}

```js
function resetChildExpirationTime(
  workInProgress: Fiber,
  renderTime: ExpirationTime,
) {
  if (renderTime !== Never && workInProgress.childExpirationTime === Never) {
    return
  }

  let newChildExpirationTime = NoWork

  // profiler 相关代码

  let child = workInProgress.child
  while (child !== null) {
    const childUpdateExpirationTime = child.expirationTime
    const childChildExpirationTime = child.childExpirationTime
    if (
      newChildExpirationTime === NoWork ||
      (childUpdateExpirationTime !== NoWork &&
        childUpdateExpirationTime < newChildExpirationTime)
    ) {
      newChildExpirationTime = childUpdateExpirationTime
    }
    if (
      newChildExpirationTime === NoWork ||
      (childChildExpirationTime !== NoWork &&
        childChildExpirationTime < newChildExpirationTime)
    ) {
      newChildExpirationTime = childChildExpirationTime
    }
    child = child.sibling
  }

  workInProgress.childExpirationTime = newChildExpirationTime
}
```

{% endmethod %}

接下去请看：

- [`completeWork`](./complete-work/complete-work.md)
- [`unwindWork`](./complete-work/unwindWork.md)
