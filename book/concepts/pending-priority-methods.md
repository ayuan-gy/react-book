{% method %}

# markPendingPriorityLevel

每次有正在执行的任务进入的时候，就会调用这个方法，用来记录最大和最小的`pending`任务

{% sample lang="js" %}

```js
export function markPendingPriorityLevel(
  root: FiberRoot,
  expirationTime: ExpirationTime,
): void {
  // If there's a gap between completing a failed root and retrying it,
  // additional updates may be scheduled. Clear `didError`, in case the update
  // is sufficient to fix the error.
  root.didError = false;

  // Update the latest and earliest pending times
  const earliestPendingTime = root.earliestPendingTime;
  if (earliestPendingTime === NoWork) {
    // No other pending updates.
    root.earliestPendingTime = root.latestPendingTime = expirationTime;
  } else {
    if (earliestPendingTime > expirationTime) {
      // This is the earliest pending update.
      root.earliestPendingTime = expirationTime;
    } else {
      const latestPendingTime = root.latestPendingTime;
      if (latestPendingTime < expirationTime) {
        // This is the latest pending update
        root.latestPendingTime = expirationTime;
      }
    }
  }
  findNextExpirationTimeToWorkOn(expirationTime, root);
}
```

{% endmethod %}

{% method %}

# markSuspendedPriorityLevel

用来设置`suspendedPriority`，分两部分

前一部分用来判断`PendingTime`是否和`suspendedTime`一样，**如果是一样代表这个任务之前调用过`markPendingPriorityLevel`，比如在`scheduleWork`的时候，那么现在这个任务被`suspended`了，则代表这个已经不是`pendingTime`了，而是`suspendedTime`了**

这里我们可以发现我们记录的永远只有两个值，最大和最小的，而寻找的时候找的基本上也是最小的

{% sample lang="js" %}

```js
export function markSuspendedPriorityLevel(
  root: FiberRoot,
  suspendedTime: ExpirationTime,
): void {
  root.didError = false;
  clearPing(root, suspendedTime);

  // First, check the known pending levels and update them if needed.
  const earliestPendingTime = root.earliestPendingTime;
  const latestPendingTime = root.latestPendingTime;
  if (earliestPendingTime === suspendedTime) {
    if (latestPendingTime === suspendedTime) {
      // Both known pending levels were suspended. Clear them.
      root.earliestPendingTime = root.latestPendingTime = NoWork;
    } else {
      // The earliest pending level was suspended. Clear by setting it to the
      // latest pending level.
      root.earliestPendingTime = latestPendingTime;
    }
  } else if (latestPendingTime === suspendedTime) {
    // The latest pending level was suspended. Clear by setting it to the
    // latest pending level.
    root.latestPendingTime = earliestPendingTime;
  }

  // Finally, update the known suspended levels.
  const earliestSuspendedTime = root.earliestSuspendedTime;
  const latestSuspendedTime = root.latestSuspendedTime;
  if (earliestSuspendedTime === NoWork) {
    // No other suspended levels.
    root.earliestSuspendedTime = root.latestSuspendedTime = suspendedTime;
  } else {
    if (earliestSuspendedTime > suspendedTime) {
      // This is the earliest suspended level.
      root.earliestSuspendedTime = suspendedTime;
    } else if (latestSuspendedTime < suspendedTime) {
      // This is the latest suspended level
      root.latestSuspendedTime = suspendedTime;
    }
  }

  findNextExpirationTimeToWorkOn(suspendedTime, root);
}
```

{% endmethod %}


{% method %}

# findNextExpirationTimeToWorkOn

首先说结论：

1. root.expirationTime是在四个里面找最小非`NoWork`的
2. root.nextExpirationTimeToWorkOn是找当前任务的下一个

```js
if (
  nextExpirationTimeToWorkOn === NoWork &&
  (completedExpirationTime === NoWork ||
    latestSuspendedTime > completedExpirationTime)
)
```
这个判断就是如果`earliestPendingTime`和`latestPingedTime`都不存在，那么选`latestSuspendedTime`，为什么不选`earliestSuspendedTime`？因为`nextExpirationTimeToWorkOn`是下一个任务，不是目前正在执行的，所以如果`earliestSuspendedTime`存在并且符合两个`pendingTime`都是`NoWork`，说明`earliestSuspendedTime`就是当前的任务过期时间，除非`earliestSuspendedTime`也是`NoWork`，那么`root.nextExpirationTimeToWorkOn`和`root.expirationTime`都是`latestSuspendedTime`。也有可能`latestSuspendedTime`也是`NoWork`，但这个就不需要判断了。


{% sample lang="js" %}

```js
function findNextExpirationTimeToWorkOn(completedExpirationTime, root) {
  const earliestSuspendedTime = root.earliestSuspendedTime;
  const latestSuspendedTime = root.latestSuspendedTime;
  const earliestPendingTime = root.earliestPendingTime;
  const latestPingedTime = root.latestPingedTime;

  // Work on the earliest pending time. Failing that, work on the latest
  // pinged time.
  let nextExpirationTimeToWorkOn =
    earliestPendingTime !== NoWork ? earliestPendingTime : latestPingedTime;

  // If there is no pending or pinged work, check if there's suspended work
  // that's lower priority than what we just completed.
  if (
    nextExpirationTimeToWorkOn === NoWork &&
    (completedExpirationTime === NoWork ||
      latestSuspendedTime > completedExpirationTime)
  ) {
    // The lowest priority suspended work is the work most likely to be
    // committed next. Let's start rendering it again, so that if it times out,
    // it's ready to commit.
    nextExpirationTimeToWorkOn = latestSuspendedTime;
  }

  let expirationTime = nextExpirationTimeToWorkOn;
  if (
    expirationTime !== NoWork &&
    earliestSuspendedTime !== NoWork &&
    earliestSuspendedTime < expirationTime
  ) {
    // Expire using the earliest known expiration time.
    expirationTime = earliestSuspendedTime;
  }

  root.nextExpirationTimeToWorkOn = nextExpirationTimeToWorkOn;
  root.expirationTime = expirationTime;
}
```

{% endmethod %}


{% method %}

# nextLatestAbsoluteTimeoutMs

这个是你设置在`placehoder`上的`delayMs`以及如果有（多层？）的情况下，计算出来的下一次根据props应该设置的过期时间的具体时间，在这个时间上会强制执行一次渲染

{% sample lang="js" %}

```js
// throwException
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


// renderDidSuspend
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

// renderRoot
if (enableSuspense && !isExpired && nextLatestAbsoluteTimeoutMs !== -1) {
  // The tree was suspended.
  const suspendedExpirationTime = expirationTime;
  markSuspendedPriorityLevel(root, suspendedExpirationTime);

  // Find the earliest uncommitted expiration time in the tree, including
  // work that is suspended. The timeout threshold cannot be longer than
  // the overall expiration.
  const earliestExpirationTime = findEarliestOutstandingPriorityLevel(
    root,
    expirationTime,
  );
  const earliestExpirationTimeMs = expirationTimeToMs(earliestExpirationTime);
  if (earliestExpirationTimeMs < nextLatestAbsoluteTimeoutMs) {
    nextLatestAbsoluteTimeoutMs = earliestExpirationTimeMs;
  }

  // Subtract the current time from the absolute timeout to get the number
  // of milliseconds until the timeout. In other words, convert an absolute
  // timestamp to a relative time. This is the value that is passed
  // to `setTimeout`.
  const currentTimeMs = expirationTimeToMs(requestCurrentTime());
  let msUntilTimeout = nextLatestAbsoluteTimeoutMs - currentTimeMs;
  msUntilTimeout = msUntilTimeout < 0 ? 0 : msUntilTimeout;

  // TODO: Account for the Just Noticeable Difference

  const rootExpirationTime = root.expirationTime;
  onSuspend(
    root,
    rootWorkInProgress,
    suspendedExpirationTime,
    rootExpirationTime,
    msUntilTimeout,
  );
}

function onSuspend(
  root: FiberRoot,
  finishedWork: Fiber,
  suspendedExpirationTime: ExpirationTime,
  rootExpirationTime: ExpirationTime,
  msUntilTimeout: number,
): void {
  root.expirationTime = rootExpirationTime;
  if (enableSuspense && msUntilTimeout === 0 && !shouldYield()) {
    // Don't wait an additional tick. Commit the tree immediately.
    root.pendingCommitExpirationTime = suspendedExpirationTime;
    root.finishedWork = finishedWork;
  } else if (msUntilTimeout > 0) {
    // Wait `msUntilTimeout` milliseconds before committing.
    root.timeoutHandle = scheduleTimeout(
      onTimeout.bind(null, root, finishedWork, suspendedExpirationTime),
      msUntilTimeout,
    );
  }
}

function onTimeout(root, finishedWork, suspendedExpirationTime) {
  if (enableSuspense) {
    // The root timed out. Commit it.
    root.pendingCommitExpirationTime = suspendedExpirationTime;
    root.finishedWork = finishedWork;
    // Read the current time before entering the commit phase. We can be
    // certain this won't cause tearing related to batching of event updates
    // because we're at the top of a timer event.
    recomputeCurrentRendererTime();
    currentSchedulerTime = currentRendererTime;

    if (enableSchedulerTracing) {
      // Don't update pending interaction counts for suspense timeouts,
      // Because we know we still need to do more work in this case.
      suspenseDidTimeout = true;
      flushRoot(root, suspendedExpirationTime);
      suspenseDidTimeout = false;
    } else {
      flushRoot(root, suspendedExpirationTime);
    }
  }
}
```

{% endmethod %}
