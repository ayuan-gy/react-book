{% method %}

# completeWork

### FunctionalComponent & FunctionalComponentLazy

什么也不做

### ClassComponent & ClassComponentLazy

如果又提供老的`context api`，则推出`context`栈

### HostRoot

推出`container`和`topLevelContextObject`

如果是初次渲染，增加`Placement`的`effectTag`，因为要把内容放入`dom`

### HostComponent

如果不是第一次渲染，则`updateComponent`

```js
updateHostComponent = function(
  current: Fiber,
  workInProgress: Fiber,
  type: Type,
  newProps: Props,
  rootContainerInstance: Container,
) {
  const oldProps = current.memoizedProps
  if (oldProps === newProps) {
    return
  }

  const instance: Instance = workInProgress.stateNode
  const currentHostContext = getHostContext()
  const updatePayload = prepareUpdate(
    instance,
    type,
    oldProps,
    newProps,
    rootContainerInstance,
    currentHostContext,
  )
  workInProgress.updateQueue = (updatePayload: any)
  if (updatePayload) {
    markUpdate(workInProgress)
  }
}
```

`prepareUpdate`做的就是我们常说的`diff`算法，调用的是[`diffProperties`](./diff-properties.md)

如果是第一次渲染，根据是否要[`hydrate`](../../basic/hydrate.md)使用不同的方法创建`dom`对象，并执行[`finalizeInitialChildren`](./finalizeInitialChildren.md)，这个方法会创建事件监听等内容。

TODO: finalizeInitialChildren.md
TODO: diffProperties.md
TODO: hydrate.md

### HostText

和`HostComponent`差不多

```js
updateHostText = function(
  current: Fiber,
  workInProgress: Fiber,
  oldText: string,
  newText: string,
) {
  // If the text differs, mark it as an update. All the work in done in commitWork.
  if (oldText !== newText) {
    markUpdate(workInProgress)
  }
}
```

`markUpdate`就是增加`Update`的`effectTag`

### HostPortal

推出`HostContainer`，`updateHostContainer`在`DOM`平台是个空方法

### ContextProvider

推出`context`栈

### 其他

都没什么操作

{% sample lang="js" %}

```js
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderExpirationTime: ExpirationTime,
): Fiber | null {
  const newProps = workInProgress.pendingProps

  switch (workInProgress.tag) {
    case FunctionalComponent:
    case FunctionalComponentLazy:
      break
    case ClassComponent: {
      const Component = workInProgress.type
      if (isLegacyContextProvider(Component)) {
        popLegacyContext(workInProgress)
      }
      break
    }
    case ClassComponentLazy: {
      const Component = getResultFromResolvedThenable(workInProgress.type)
      if (isLegacyContextProvider(Component)) {
        popLegacyContext(workInProgress)
      }
      break
    }
    case HostRoot: {
      popHostContainer(workInProgress)
      popTopLevelLegacyContextObject(workInProgress)
      const fiberRoot = (workInProgress.stateNode: FiberRoot)
      if (fiberRoot.pendingContext) {
        fiberRoot.context = fiberRoot.pendingContext
        fiberRoot.pendingContext = null
      }
      if (current === null || current.child === null) {
        popHydrationState(workInProgress)
        workInProgress.effectTag &= ~Placement
      }
      updateHostContainer(workInProgress)
      break
    }
    case HostComponent: {
      popHostContext(workInProgress)
      const rootContainerInstance = getRootHostContainer()
      const type = workInProgress.type
      if (current !== null && workInProgress.stateNode != null) {
        updateHostComponent(
          current,
          workInProgress,
          type,
          newProps,
          rootContainerInstance,
        )

        if (current.ref !== workInProgress.ref) {
          markRef(workInProgress)
        }
      } else {
        if (!newProps) {
          invariant(
            workInProgress.stateNode !== null,
            'We must have new props for new mounts. This error is likely ' +
              'caused by a bug in React. Please file an issue.',
          )
          // This can happen when we abort work.
          break
        }

        const currentHostContext = getHostContext()
        let wasHydrated = popHydrationState(workInProgress)
        if (wasHydrated) {
          if (
            prepareToHydrateHostInstance(
              workInProgress,
              rootContainerInstance,
              currentHostContext,
            )
          ) {
            markUpdate(workInProgress)
          }
        } else {
          let instance = createInstance(
            type,
            newProps,
            rootContainerInstance,
            currentHostContext,
            workInProgress,
          )

          appendAllChildren(instance, workInProgress)

          if (
            finalizeInitialChildren(
              instance,
              type,
              newProps,
              rootContainerInstance,
              currentHostContext,
            )
          ) {
            markUpdate(workInProgress)
          }
          workInProgress.stateNode = instance
        }

        if (workInProgress.ref !== null) {
          // If there is a ref on a host node we need to schedule a callback
          markRef(workInProgress)
        }
      }
      break
    }
    case HostText: {
      let newText = newProps
      if (current && workInProgress.stateNode != null) {
        const oldText = current.memoizedProps
        // If we have an alternate, that means this is an update and we need
        // to schedule a side-effect to do the updates.
        updateHostText(current, workInProgress, oldText, newText)
      } else {
        if (typeof newText !== 'string') {
          invariant(
            workInProgress.stateNode !== null,
            'We must have new props for new mounts. This error is likely ' +
              'caused by a bug in React. Please file an issue.',
          )
          // This can happen when we abort work.
        }
        const rootContainerInstance = getRootHostContainer()
        const currentHostContext = getHostContext()
        let wasHydrated = popHydrationState(workInProgress)
        if (wasHydrated) {
          if (prepareToHydrateHostTextInstance(workInProgress)) {
            markUpdate(workInProgress)
          }
        } else {
          workInProgress.stateNode = createTextInstance(
            newText,
            rootContainerInstance,
            currentHostContext,
            workInProgress,
          )
        }
      }
      break
    }
    case ForwardRef:
    case ForwardRefLazy:
      break
    case PlaceholderComponent:
      break
    case Fragment:
      break
    case Mode:
      break
    case Profiler:
      break
    case HostPortal:
      popHostContainer(workInProgress)
      updateHostContainer(workInProgress)
      break
    case ContextProvider:
      // Pop provider fiber
      popProvider(workInProgress)
      break
    case ContextConsumer:
      break
    // Error cases
    case IndeterminateComponent:
      invariant(
        false,
        'An indeterminate component should have become determinate before ' +
          'completing. This error is likely caused by a bug in React. Please ' +
          'file an issue.',
      )
    // eslint-disable-next-line no-fallthrough
    default:
      invariant(
        false,
        'Unknown unit of work tag. This error is likely caused by a bug in ' +
          'React. Please file an issue.',
      )
  }

  return null
}
```

{% endmethod %}

下一节请看：

- [`diffProperties`](./diff-properties.md)
