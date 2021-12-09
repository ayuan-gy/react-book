{% method %}

# updateTimeoutComponent

`enableSuspense`目前来看是`false`，可以查看`ReactFeatureFlags`

看注释应该是跟17年底Dan分享的`React Async Components`有关，可以看一下[这个视频](https://reactjs.org/blog/2018/03/01/sneak-peek-beyond-react-16.html)

{% sample lang="js" %}

```js
function updateTimeoutComponent(current, workInProgress, renderExpirationTime) {
  if (enableSuspense) {
    const nextProps = workInProgress.pendingProps;
    const prevProps = workInProgress.memoizedProps;

    const prevDidTimeout = workInProgress.memoizedState;

    // Check if we already attempted to render the normal state. If we did,
    // and we timed out, render the placeholder state.
    const alreadyCaptured =
      (workInProgress.effectTag & DidCapture) === NoEffect;
    const nextDidTimeout = !alreadyCaptured;

    if (hasLegacyContextChanged()) {
      // Normally we can bail out on props equality but if context has changed
      // we don't do the bailout and we have to reuse existing props instead.
    } else if (nextProps === prevProps && nextDidTimeout === prevDidTimeout) {
      return bailoutOnAlreadyFinishedWork(current, workInProgress);
    }

    const render = nextProps.children;
    const nextChildren = render(nextDidTimeout);
    workInProgress.memoizedProps = nextProps;
    workInProgress.memoizedState = nextDidTimeout;
    reconcileChildren(current, workInProgress, nextChildren);
    return workInProgress.child;
  } else {
    return null;
  }
}
```

{% endmethod %}