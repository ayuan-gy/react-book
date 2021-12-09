# PendingPriority

### earliestPendingTime

### latestPendingTime



# CommittedPriority

### earliestPendingTime

### latestPendingTime



# SuspendedPriority

### earliestSuspendedTime

### latestSuspendedTime


# PingedPriority

### latestPingedTime




##### currentRendererTime

```js
function recomputeCurrentRendererTime() {
  const currentTimeMs = now() - originalStartTimeMs;
  currentRendererTime = msToExpirationTime(currentTimeMs);
}

const UNIT_SIZE = 10;
const MAGIC_NUMBER_OFFSET = 2;
export function msToExpirationTime(ms: number): ExpirationTime {
  // Always add an offset so that we don't clash with the magic number for NoWork.
  return ((ms / UNIT_SIZE) | 0) + MAGIC_NUMBER_OFFSET;
}
```

##### callbackExpirationTime

用来记录上次异步调度的超时时间，在`scheduleCallbackWithExpirationTime`中设置成`expirationTime`，在回调`performWork`开始时设置成`NoWork`，这是为了防止连续出现两次回调，

`callbackExpirationTime`中的判断，如果第二次调用的`expirationTime`比第一次的超时时间小，则要撤销上一次回调，使用这次的`expirationTime`进行回调

```js
if (callbackExpirationTime !== NoWork) {
  // A callback is already scheduled. Check its expiration time (timeout).
  if (expirationTime > callbackExpirationTime) {
    // Existing callback has sufficient timeout. Exit.
    return;
  } else {
    if (callbackID !== null) {
      // Existing callback has insufficient timeout. Cancel and schedule a
      // new one.
      cancelDeferredCallback(callbackID);
    }
  }
  // The request callback timer is already running. Don't start a new one.
} else {
  startRequestCallbackTimer();
}
```

##### nextLatestAbsoluteTimeoutMs

这个时间是一旦遇到`placeholder`并且设置了`delayMs`，在这个时间结束之前，会设置到`nextLatestAbsoluteTimeoutMs`

在`resetStack`的时候被设置为`-1`

在`renderRoot`的时候
```js
if (enableSuspense && !isExpired && nextLatestAbsoluteTimeoutMs !== -1) {
  // The tree was suspended.
  const suspendedExpirationTime = expirationTime;
  markSuspendedPriorityLevel(root, suspendedExpirationTime);

  const earliestExpirationTime = findEarliestOutstandingPriorityLevel(
    root,
    expirationTime,
  );
  const earliestExpirationTimeMs = expirationTimeToMs(earliestExpirationTime);
  if (earliestExpirationTimeMs < nextLatestAbsoluteTimeoutMs) {
    nextLatestAbsoluteTimeoutMs = earliestExpirationTimeMs;
  }

  // 。。。。。。
}
```

在`renderDidSuspend`的时候，这个方法在`throwException`里面调用，也就是只有真的抛出一个`promise`的时候才会调用

可以认为一旦能执行到`renderDidSuspend`说明`nextLatestAbsoluteTimeoutMs`肯定是大于0的，并且`placeholder`还没有过期（不确定过期在哪里设置，也不确定是否超过了`delayMs`就是过期）
```js
function renderDidSuspend(
  root: FiberRoot,
  absoluteTimeoutMs: number,
  suspendedTime: ExpirationTime,
) {
  // Schedule the timeout.
  if (
    absoluteTimeoutMs >= 0 &&
    nextLatestAbsoluteTimeoutMs < absoluteTimeoutMs
  ) {
    nextLatestAbsoluteTimeoutMs = absoluteTimeoutMs;
  }
}

// 关键代码
const didTimeout = workInProgress.memoizedState;
if (!didTimeout) {
  if ((workInProgress.mode & StrictMode) === NoEffect) {
    // ...
    return
  }
  
  let absoluteTimeoutMs;
  if (earliestTimeoutMs === -1) {
    absoluteTimeoutMs = maxSigned31BitInt;
  } else {
    if (startTimeMs === -1) {
      const earliestExpirationTime = findEarliestOutstandingPriorityLevel(
        root,
        renderExpirationTime,
      );
      const earliestExpirationTimeMs = expirationTimeToMs(
        earliestExpirationTime,
      );
      startTimeMs = earliestExpirationTimeMs - LOW_PRIORITY_EXPIRATION;
    }
    absoluteTimeoutMs = startTimeMs + earliestTimeoutMs;
  }

  renderDidSuspend(root, absoluteTimeoutMs, renderExpirationTime);
  // ...
}

```

计算`earliestTimeoutMs`，可以认为是在`PlaceholderComponent`还没过期的时候的`delayMs`这个`props`的值，并且是向上找到最小的`delayMs`
```js
let earliestTimeoutMs = -1;
let startTimeMs = -1;
do {
  if (workInProgress.tag === PlaceholderComponent) {
    const current = workInProgress.alternate;
    if (
      current !== null &&
      current.memoizedState === true &&
      current.stateNode !== null
    ) {
      // Reached a placeholder that already timed out. Each timed out
      // placeholder acts as the root of a new suspense boundary.

      // Use the time at which the placeholder timed out as the start time
      // for the current render.
      const timedOutAt = current.stateNode.timedOutAt;
      startTimeMs = expirationTimeToMs(timedOutAt);

      // Do not search any further.
      break;
    }
    let timeoutPropMs = workInProgress.pendingProps.delayMs;
    if (typeof timeoutPropMs === 'number') {
      if (timeoutPropMs <= 0) {
        earliestTimeoutMs = 0;
      } else if (
        earliestTimeoutMs === -1 ||
        timeoutPropMs < earliestTimeoutMs
      ) {
        earliestTimeoutMs = timeoutPropMs;
      }
    }
  }
```

`startTimeMs`则是记录已经过期的`placeholder`的`timedOutAt`属性`current.memoizedState === true`代表已经过期。一旦找到一个过期的则直接`break`
```js
const timedOutAt = current.stateNode.timedOutAt;
startTimeMs = expirationTimeToMs(timedOutAt);
```


##### nextRenderDidError

在抛出错误并且不是`Promise`的时候会设置成ture，代表真的有错误