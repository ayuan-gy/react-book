# react scheduler

`react-scheduler/src/ReactScheduler.js`里面包含了主要的React调度相关的代码，通过控制每一帧中间的时间来让浏览器有足够的空闲事件进行渲染工作，这里让我们分析一下里面的代码。

{% method %}

# scheduleWork

这个方法接受两个参数

1. callback就是需要调度的任务
2. options通过flow的类型可以看出是一个只有timeout参数的对象，这个timeout是指多久没执行这个callback就代表已经超时了，需要尽快执行

```js
let timeoutTime = -1;
if (options != null && typeof options.timeout === 'number') {
  timeoutTime = now() + options.timeout;
}
if (
  nextSoonestTimeoutTime === -1 ||
  (timeoutTime !== -1 && timeoutTime < nextSoonestTimeoutTime)
) {
  nextSoonestTimeoutTime = timeoutTime;
}
```

这边就是初始化一些变量，传入的`timeout`只是一个时间长度，这边要换算成具体的事件点。`nextSoonestTimeoutTime`是这里的一个全局变量，如果新的任务的超时事件比目前最近的还要小，那么设置当前的为最小时间

```js
const scheduledCallbackConfig: CallbackConfigType = {
  scheduledCallback: callback,
  timeoutTime,
  prev: null,
  next: null,
};
```
创建`callbackConfig`对象，可以认为他就是在**Fiber调度**时的`deadline`，在`ReactFiberScheduler`中的`shouldYield`方法中会用他来判断任务执行是否超时，需要暂停

```js
if (headOfPendingCallbacksLinkedList === null) {
  // Make this callback the head and tail of our list
  headOfPendingCallbacksLinkedList = scheduledCallbackConfig;
  tailOfPendingCallbacksLinkedList = scheduledCallbackConfig;
} else {
  // Add latest callback as the new tail of the list
  scheduledCallbackConfig.prev = tailOfPendingCallbacksLinkedList;
  // renaming for clarity
  const oldTailOfPendingCallbacksLinkedList = tailOfPendingCallbacksLinkedList;
  if (oldTailOfPendingCallbacksLinkedList !== null) {
    oldTailOfPendingCallbacksLinkedList.next = scheduledCallbackConfig;
  }
  tailOfPendingCallbacksLinkedList = scheduledCallbackConfig;
}
```

`headOfPendingCallbacksLinkedList`和`tailOfPendingCallbacksLinkedList`又是一个链表结构，每一个节点包含`prev`和`next`迎来链接上一个任务和下一个任务，在这里就是执行把当前任务插入链表的操作

最后判断`isAnimationFrameScheduled`是否为`false`，如果是则主动调用他，如果不是说明已经在操作`CallbacksLinkedList`，那么我们已经插入了任务，自然而然会执行到。

{% sample lang="js" %}

```js
scheduleWork = function(
  callback: FrameCallbackType,
  options?: {timeout: number},
): CallbackIdType /* CallbackConfigType */ {
  let timeoutTime = -1;
  if (options != null && typeof options.timeout === 'number') {
    timeoutTime = now() + options.timeout;
  }
  if (
    nextSoonestTimeoutTime === -1 ||
    (timeoutTime !== -1 && timeoutTime < nextSoonestTimeoutTime)
  ) {
    nextSoonestTimeoutTime = timeoutTime;
  }

  const scheduledCallbackConfig: CallbackConfigType = {
    scheduledCallback: callback,
    timeoutTime,
    prev: null,
    next: null,
  };
  if (headOfPendingCallbacksLinkedList === null) {
    // Make this callback the head and tail of our list
    headOfPendingCallbacksLinkedList = scheduledCallbackConfig;
    tailOfPendingCallbacksLinkedList = scheduledCallbackConfig;
  } else {
    // Add latest callback as the new tail of the list
    scheduledCallbackConfig.prev = tailOfPendingCallbacksLinkedList;
    // renaming for clarity
    const oldTailOfPendingCallbacksLinkedList = tailOfPendingCallbacksLinkedList;
    if (oldTailOfPendingCallbacksLinkedList !== null) {
      oldTailOfPendingCallbacksLinkedList.next = scheduledCallbackConfig;
    }
    tailOfPendingCallbacksLinkedList = scheduledCallbackConfig;
  }

  if (!isAnimationFrameScheduled) {
    // If rAF didn't already schedule one, we need to schedule a frame.
    // TODO: If this rAF doesn't materialize because the browser throttles, we
    // might want to still have setTimeout trigger scheduleWork as a backup to ensure
    // that we keep performing work.
    isAnimationFrameScheduled = true;
    localRequestAnimationFrame(animationTick);
  }
  return scheduledCallbackConfig;
};
```

{% endmethod %}

{% method %}

# animationTick

`localRequestAnimationFrame`就是`requestAnimationFrame`

`localRequestAnimationFrame(animationTick)`就意味着尽快调用`animationTick`

`rafTime`是`requestAnimationFram`传递过来的，值是`performance.now()`

这边有几个全局变量：

1. frameDeadline 一帧最晚完成的时间
2. activeFrameTime 每一帧可用的时间
3. previousFrameTime 上一帧最晚完成时间

这边做的最主要的一个事情是动态得调整`activeFrameTime`的值，也就是在保持最低30帧的情况下，可以提高帧数。`activeFrameTime`初始是`33`也就是一秒30帧对应的每帧的时间。

试想如果我们连续间隔小于`33ms`的时间调用`animationTick`，那么`rafTime - frameDeadline + activeFrameTime`肯定是小于`activeFrameTime`的，因为`frameDeadline = rafTime（上一个） + activeFrameTime`，也就是当前的`rafTime`如果正好是上次`33ms`之后调用的，那么`nextFrameTime`正好等于`activeFrameTime`。

`previousFrameTime`存在的意义是要保证连续两次出现`nextFrameTime`小于当前`activeFrameTime`才会调低每帧时间，而不会因为一些意外情况就随意调整

可见的是动态调低的条件是很苛刻的，要连续两次，对于一般的应用来说，并不会太密集的运算和动画出现，可能都很少会出现连续调用`scheduleWork`的情况

```js
if (!isIdleScheduled) {
  isIdleScheduled = true;
  window.postMessage(messageKey, '*');
}
```
最后这里要调用`idelTick`，不清楚为什么要用`postMessage`来触发，看注释是因为要等当前的`repaint`结束之后才调用？`postMessage`属于`task`

TODO: 为啥要用`postMessage`触发`idelTick`，`postMessage`的执行时机

{% sample lang="js" %}

```js
const animationTick = function(rafTime) {
  isAnimationFrameScheduled = false;
  let nextFrameTime = rafTime - frameDeadline + activeFrameTime;
  if (
    nextFrameTime < activeFrameTime &&
    previousFrameTime < activeFrameTime
  ) {
    if (nextFrameTime < 8) {
      // Defensive coding. We don't support higher frame rates than 120hz.
      // If we get lower than that, it is probably a bug.
      nextFrameTime = 8;
    }

    activeFrameTime =
      nextFrameTime < previousFrameTime ? previousFrameTime : nextFrameTime;
  } else {
    previousFrameTime = nextFrameTime;
  }
  frameDeadline = rafTime + activeFrameTime;
  if (!isIdleScheduled) {
    isIdleScheduled = true;
    window.postMessage(messageKey, '*');
  }
};
```

{% endmethod %}

{% method %}

# IdelTick

首先执行`callTimedOutCallbacks`，把已经处于超时状态的任务先执行了。

然后进入循环，依次执行`callbackLinkedList`中的每一项，并且保持在最晚时间之内

清空队列或者到`deadline`之后，再次判断是否还有任务剩余，如果有，再次进入`animationTick`

{% sample lang="js" %}

```js
// We use the postMessage trick to defer idle work until after the repaint.
const messageKey =
  '__reactIdleCallback$' +
  Math.random()
    .toString(36)
    .slice(2);
const idleTick = function(event) {
  if (event.source !== window || event.data !== messageKey) {
    return;
  }
  isIdleScheduled = false;

  if (headOfPendingCallbacksLinkedList === null) {
    return;
  }

  // First call anything which has timed out, until we have caught up.
  callTimedOutCallbacks();

  let currentTime = now();
  // Next, as long as we have idle time, try calling more callbacks.
  while (
    frameDeadline - currentTime > 0 &&
    headOfPendingCallbacksLinkedList !== null
  ) {
    const latestCallbackConfig = headOfPendingCallbacksLinkedList;
    frameDeadlineObject.didTimeout = false;
    // callUnsafely will remove it from the head of the linked list
    callUnsafely(latestCallbackConfig, frameDeadlineObject);
    currentTime = now();
  }
  if (headOfPendingCallbacksLinkedList !== null) {
    if (!isAnimationFrameScheduled) {
      // Schedule another animation callback so we retry later.
      isAnimationFrameScheduled = true;
      localRequestAnimationFrame(animationTick);
    }
  }
};
// Assumes that we have addEventListener in this environment. Might need
// something better for old IE.
window.addEventListener('message', idleTick, false);
```

{% endmethod %}

{% method %}

# callTimedOutCallbacks

这个没啥好说的，就是一次判断链表每一项，如果`timeoutTime <= currentTime`就加入到`timeOutCallbacks`数组中，然后一次调用他们。

{% sample lang="js" %}

```js
const callTimedOutCallbacks = function() {
  if (headOfPendingCallbacksLinkedList === null) {
    return;
  }

  const currentTime = now();
  if (nextSoonestTimeoutTime === -1 || nextSoonestTimeoutTime > currentTime) {
    // We know that none of them have timed out yet.
    return;
  }

  let updatedNextSoonestTimeoutTime = -1; // we will update nextSoonestTimeoutTime below
  const timedOutCallbacks = [];

  // iterate once to find timed out callbacks and find nextSoonestTimeoutTime
  let currentCallbackConfig = headOfPendingCallbacksLinkedList;
  while (currentCallbackConfig !== null) {
    const timeoutTime = currentCallbackConfig.timeoutTime;
    if (timeoutTime !== -1 && timeoutTime <= currentTime) {
      // it has timed out!
      timedOutCallbacks.push(currentCallbackConfig);
    } else {
      if (
        timeoutTime !== -1 &&
        (updatedNextSoonestTimeoutTime === -1 ||
          timeoutTime < updatedNextSoonestTimeoutTime)
      ) {
        updatedNextSoonestTimeoutTime = timeoutTime;
      }
    }
    currentCallbackConfig = currentCallbackConfig.next;
  }

  if (timedOutCallbacks.length > 0) {
    frameDeadlineObject.didTimeout = true;
    for (let i = 0, len = timedOutCallbacks.length; i < len; i++) {
      callUnsafely(timedOutCallbacks[i], frameDeadlineObject);
    }
  }

  // NOTE: we intentionally wait to update the nextSoonestTimeoutTime until
  // after successfully calling any timed out callbacks.
  nextSoonestTimeoutTime = updatedNextSoonestTimeoutTime;
};
```

{% endmethod %}

{% method %}

# callUnsafely

`arg`就是`frameDeadlineObject`也就是`deadline`，结构如下

```js
const frameDeadlineObject: Deadline = {
  didTimeout: false,
  timeRemaining() {
    const remaining = frameDeadline - now();
    return remaining > 0 ? remaining : 0;
  },
};
```

不论最终是否有`catch`到错误，都会取消当前任务，并且如果没有正常结束，则会再次进入`idelTick`

{% sample lang="js" %}

```js
const callUnsafely = function(
  callbackConfig: CallbackConfigType,
  arg: Deadline,
) {
  const callback = callbackConfig.scheduledCallback;
  let finishedCalling = false;
  try {
    callback(arg);
    finishedCalling = true;
  } finally {
    // always remove it from linked list
    cancelScheduledWork(callbackConfig);

    if (!finishedCalling) {
      // an error must have been thrown
      isIdleScheduled = true;
      window.postMessage(messageKey, '*');
    }
  }
};
```

{% endmethod %}