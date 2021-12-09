# Time About

{% method %}

### recakcukateCurrentTime

其中`originalStartTimeMs`在页面加载之后就运行为`now()`，是一个固定值

也就是说这里计算出了一个距离页面初始化完成之后的一个事件间隔值

`(ms / UNIT_SIZE | 0)`是除以10然后再取整数，然后再加二，加二是为了防止除以10之后出现等于`NoWork`也就是0的情况。

由此可知这个计算公式得到的就是页面加载完成之后到现在为止的毫秒数处以10再加2

{% sample lang="js" %}

```js
function recalculateCurrentTime() {
  // Subtract initial time so it fits inside 32bits
  mostRecentCurrentTimeMs = now() - originalStartTimeMs;
  mostRecentCurrentTime = msToExpirationTime(mostRecentCurrentTimeMs);
  return mostRecentCurrentTime;
}

var UNIT_SIZE = 10;
var MAGIC_NUMBER_OFFSET = 2;
function msToExpirationTime(ms) {
  // Always add an offset so that we don't clash with the magic number for NoWork.
  return (ms / UNIT_SIZE | 0) + MAGIC_NUMBER_OFFSET;
}
```

{% endmethod %}

{% method %}

### computeExpirationForFiber

`expirationContext`默认是`NoWork`，在调用`defferedUpdate`和`syncUpdate`的时候分别会设置成`computeAsyncExpiration(currentTime)`和`Sync`分别代表这是`Async Update`和`Sync Update`

`nextRenderExpirationTime`默认是`NoWork`，在`renderRoot`的时候会被设置为`nextRenderExpirationTime = expirationTime`，并且每次都在`resetStack`之后运行，所以并不会出现该值还是上一次任务的事件的情况

如果`expirationContext`不等于`NoWork`说明是通过特定方法调用指定的更新方式，就让过期直接等于context

然后如果有工作正在进行，并且正处于`commit`阶段，那么设置超时事件为`Sync`，如果不是则是`nextRenderExpirationTime`，也就是上一次任务的过期时间。

如果都不是，那么判断是否是`AsyncMode`。如果是并且`isBatchingInteractiveUpdates`为`true`，则过期时间为`computeInteractiveExpiration(currentTime)`，如果不为true则等于`computeAsyncExpiration(currentTime)`

`computeInteractiveExpiration`和`computeAsyncExpiration`都会调用`computeExpirationBucket`，他们的区别可以看[这里](#compiteExpiration)

```js
if (isBatchingInteractiveUpdates) {
  // This is an interactive update. Keep track of the lowest pending
  // interactive expiration time. This allows us to synchronously flush
  // all interactive updates when needed.
  if (lowestPendingInteractiveExpirationTime === NoWork || expirationTime > lowestPendingInteractiveExpirationTime) {
    lowestPendingInteractiveExpirationTime = expirationTime;
  }
}
```

这段代码是用来记录最小`interactiveUpdates`的`expirationTime`。

`lowestPendingInteractiveExpirationTime`的作用是在一下次`interactiveUpdates`的时候使用，并且使用之后会置为空

{% sample lang="js" %}

```js
function computeExpirationForFiber(currentTime, fiber) {
  var expirationTime = void 0;
  if (expirationContext !== NoWork) {
    // An explicit expiration context was set;
    expirationTime = expirationContext;
  } else if (isWorking) {
    if (isCommitting$1) {
      // Updates that occur during the commit phase should have sync priority
      // by default.
      expirationTime = Sync;
    } else {
      // Updates during the render phase should expire at the same time as
      // the work that is being rendered.
      expirationTime = nextRenderExpirationTime;
    }
  } else {
    // No explicit expiration context was set, and we're not currently
    // performing work. Calculate a new expiration time.
    if (fiber.mode & AsyncMode) {
      if (isBatchingInteractiveUpdates) {
        // This is an interactive update
        expirationTime = computeInteractiveExpiration(currentTime);
      } else {
        // This is an async update
        expirationTime = computeAsyncExpiration(currentTime);
      }
    } else {
      // This is a sync update
      expirationTime = Sync;
    }
  }
  if (isBatchingInteractiveUpdates) {
    // This is an interactive update. Keep track of the lowest pending
    // interactive expiration time. This allows us to synchronously flush
    // all interactive updates when needed.
    if (lowestPendingInteractiveExpirationTime === NoWork || expirationTime > lowestPendingInteractiveExpirationTime) {
      lowestPendingInteractiveExpirationTime = expirationTime;
    }
  }
  return expirationTime;
}

function interactiveUpdates<A, B, R>(fn: (A, B) => R, a: A, b: B): R {
  if (isBatchingInteractiveUpdates) {
    return fn(a, b);
  }
  // If there are any pending interactive updates, synchronously flush them.
  // This needs to happen before we read any handlers, because the effect of
  // the previous event may influence which handlers are called during
  // this event.
  if (
    !isBatchingUpdates &&
    !isRendering &&
    lowestPendingInteractiveExpirationTime !== NoWork
  ) {
    // Synchronously flush pending interactive updates.
    performWork(lowestPendingInteractiveExpirationTime, false, null);
    lowestPendingInteractiveExpirationTime = NoWork;
  }
  const previousIsBatchingInteractiveUpdates = isBatchingInteractiveUpdates;
  const previousIsBatchingUpdates = isBatchingUpdates;
  isBatchingInteractiveUpdates = true;
  isBatchingUpdates = true;
  try {
    return fn(a, b);
  } finally {
    isBatchingInteractiveUpdates = previousIsBatchingInteractiveUpdates;
    isBatchingUpdates = previousIsBatchingUpdates;
    if (!isBatchingUpdates && !isRendering) {
      performSyncWork();
    }
  }
}

function flushInteractiveUpdates() {
  if (!isRendering && lowestPendingInteractiveExpirationTime !== NoWork) {
    // Synchronously flush pending interactive updates.
    performWork(lowestPendingInteractiveExpirationTime, false, null);
    lowestPendingInteractiveExpirationTime = NoWork;
  }
}
```

{% endmethod %}

{% method %}

# computeAsyncExpiration & computeInteractiveExpiration {#compiteExpiration}

`computeInteractiveExpiration`和`computeAsyncExpiration`都会调用`computeExpirationBucket`

唯一的区别是参数不同

* `computeInteractiveExpiration`是`currentTime, 150, 100`
* `computeExpirationBucket`是`currentTime, 5000, 500`

`computeExpirationBucket`的计算结果

1. 输入currentTime 5000 250 得到currentTime + 502
2. 输入currentTime 500 100 得到currentTime + 52
3. 输入currentTime 150 100 得到currentTime + 22

可见这是一个固定值，也就是在`502`毫秒内结束，或者在`22`毫秒内结束，52是只有在开发时才会出现的。

{% sample lang="js" %}

```js
function computeAsyncExpiration(currentTime: ExpirationTime) {
  const expirationMs = 5000;
  const bucketSizeMs = 250;
  return computeExpirationBucket(currentTime, expirationMs, bucketSizeMs);
}

function computeInteractiveExpiration(currentTime: ExpirationTime) {
  let expirationMs;
  if (__DEV__) {
    expirationMs = 500;
  } else {
    expirationMs = 150;
  }
  const bucketSizeMs = 100;
  return computeExpirationBucket(currentTime, expirationMs, bucketSizeMs);
}

function ceiling(num: number, precision: number): number {
  return (((num / precision) | 0) + 1) * precision;
}

export function computeExpirationBucket(
  currentTime: ExpirationTime,
  expirationInMs: number,
  bucketSizeMs: number,
): ExpirationTime {
  return (
    MAGIC_NUMBER_OFFSET +
    ceiling(
      currentTime - MAGIC_NUMBER_OFFSET + expirationInMs / UNIT_SIZE,
      bucketSizeMs / UNIT_SIZE,
    )
  );
}
```

{% endmethod %}