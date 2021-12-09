{% method %}

# unwindWork

跟[`completeWork`](./complete-work.md)其实差不多，最主要的区别是处理了`ShouldCapture`的副作用，这是在`throwException`的时候被添加的，只针对于有能力处理异常的`ClassComponent`或者直接到`HostRoot`。

注意处理完`ShouldCapture`的组件都会增加`DidCapture`副作用并返回其`Fiber`对象，返回`Fiber`意味着在`completeUnitOfWork`中的`next`有值，则意味着当前组件需要重新走更新（`beginWork`）流程

{% sample lang="js" %}

```js
function unwindWork(
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
) {
  switch (workInProgress.tag) {
    case ClassComponent: {
      const Component = workInProgress.type
      if (isLegacyContextProvider(Component)) {
        popLegacyContext(workInProgress)
      }
      const effectTag = workInProgress.effectTag
      if (effectTag & ShouldCapture) {
        workInProgress.effectTag = (effectTag & ~ShouldCapture) | DidCapture
        return workInProgress
      }
      return null
    }
    case HostRoot: {
      popHostContainer(workInProgress)
      popTopLevelLegacyContextObject(workInProgress)
      const effectTag = workInProgress.effectTag
      invariant(
        (effectTag & DidCapture) === NoEffect,
        'The root failed to unmount after an error. This is likely a bug in ' +
          'React. Please file an issue.',
      )
      workInProgress.effectTag = (effectTag & ~ShouldCapture) | DidCapture
      return workInProgress
    }
    case HostComponent: {
      popHostContext(workInProgress)
      return null
    }
    case SuspenseComponent: {
      const effectTag = workInProgress.effectTag
      if (effectTag & ShouldCapture) {
        workInProgress.effectTag = (effectTag & ~ShouldCapture) | DidCapture
        // Captured a suspense effect. Set the boundary's `alreadyCaptured`
        // state to true so we know to render the fallback.
        const current = workInProgress.alternate
        const currentState: SuspenseState | null =
          current !== null ? current.memoizedState : null
        let nextState: SuspenseState | null = workInProgress.memoizedState
        if (nextState === null) {
          // No existing state. Create a new object.
          nextState = {
            alreadyCaptured: true,
            didTimeout: false,
            timedOutAt: NoWork,
          }
        } else if (currentState === nextState) {
          // There is an existing state but it's the same as the current tree's.
          // Clone the object.
          nextState = {
            alreadyCaptured: true,
            didTimeout: nextState.didTimeout,
            timedOutAt: nextState.timedOutAt,
          }
        } else {
          // Already have a clone, so it's safe to mutate.
          nextState.alreadyCaptured = true
        }
        workInProgress.memoizedState = nextState
        // Re-render the boundary.
        return workInProgress
      }
      return null
    }
    case HostPortal:
      popHostContainer(workInProgress)
      return null
    case ContextProvider:
      popProvider(workInProgress)
      return null
    default:
      return null
  }
}
```

{% endmethod %}
