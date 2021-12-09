# updateQueue

> 主要代码再`ReactUpdateQueue.js`

`updateQueue`是React中承载**更新任务**的一个队列，所谓更新任务也就是`setState`和`forceUpdate`等API调用时会创建的一个`Update`对象，用于记录这次更新的发起者，更新的内容等。这里需要关注的是以下几个方法：

- `enqueueUpdate`

{% method %}

# enqueueUpdate

要读懂这个方法，我们需要关注这个方法在什么时候会被调用：

- 在调用`setState`、`forceUpdate`等主动创建更新的API的时候
- 在`NewContext`中更新`context`的值的时候
- 在调用`ReactDOM.render`进行初次渲染的时候
- 在有错误被捕获的时候

我们可以看到这些都是会影响我们的UI渲染的内容，同时就是他们都会创建`Update`对象。再根据函数定义我们可以看出来，他接受的两个参数：

- Fiber
- Update

`Fiber`就是产生更新的节点，`Update`就是产生的更新对象，那么这个函数就很明确了：

**用来把创建的更新记录在对应节点的Fiber对象上**

接下去我们再来看他的实现，就显得比较容易理解了：

`queue1`和`queue2`分别对应的是`current`和`workInProgress`对象上面的`queue`，根据这个函数的最终结果我们是可以得出一个结论的，那就是：**`current`和`workInProgress`上面的`queue`对象是不同的，但是他们记录的`update`链表是一样的**。为什么要这么做呢？因为后面我们讲到`Suspense`的时候，我们会讲到在React中更新任务是可以被挂起甚至被抛弃的，而在更新过程中我们又会修改`queue`对象，如果`current`和`workInProgress`引用的`queue`对象相同，**那么就会导致被舍弃的任务影响到正常的更新队列。**如果你目前无法理解这些，你现在只需要知道他们是两个不同的对象，但是新的`Update`都会放到这两个队列上就可以了。

而这个函数做的事情，就是获取`current`和`workInProgress`对象上的`queue`，然后把新的`Update`放到这两个`queue`的最后。

{% sample lang="js" %}

```js
export function enqueueUpdate<State>(fiber: Fiber, update: Update<State>) {
  // Update queues are created lazily.
  const alternate = fiber.alternate;
  let queue1;
  let queue2;
  if (alternate === null) {
    // There's only one fiber.
    queue1 = fiber.updateQueue;
    queue2 = null;
    if (queue1 === null) {
      queue1 = fiber.updateQueue = createUpdateQueue(fiber.memoizedState);
    }
  } else {
    // There are two owners.
    queue1 = fiber.updateQueue;
    queue2 = alternate.updateQueue;
    if (queue1 === null) {
      if (queue2 === null) {
        // Neither fiber has an update queue. Create new ones.
        queue1 = fiber.updateQueue = createUpdateQueue(fiber.memoizedState);
        queue2 = alternate.updateQueue = createUpdateQueue(
          alternate.memoizedState,
        );
      } else {
        // Only one fiber has an update queue. Clone to create a new one.
        queue1 = fiber.updateQueue = cloneUpdateQueue(queue2);
      }
    } else {
      if (queue2 === null) {
        // Only one fiber has an update queue. Clone to create a new one.
        queue2 = alternate.updateQueue = cloneUpdateQueue(queue1);
      } else {
        // Both owners have an update queue.
      }
    }
  }
  if (queue2 === null || queue1 === queue2) {
    // There's only a single queue.
    appendUpdateToQueue(queue1, update);
  } else {
    if (queue1.lastUpdate === null || queue2.lastUpdate === null) {
      // One of the queues is not empty. We must add the update to both queues.
      appendUpdateToQueue(queue1, update);
      appendUpdateToQueue(queue2, update);
    } else {
      appendUpdateToQueue(queue1, update);
      // But we still need to update the `lastUpdate` pointer of queue2.
      queue2.lastUpdate = update;
    }
  }
}
```

{% endmethod %}
